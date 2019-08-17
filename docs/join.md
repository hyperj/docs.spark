# Join

```antlrv4
joinRelation
    : (joinType) JOIN right=relationPrimary joinCriteria?
    | NATURAL joinType JOIN right=relationPrimary
    ;

joinType
    : INNER?
    | CROSS
    | LEFT OUTER?
    | LEFT SEMI
    | RIGHT OUTER?
    | FULL OUTER?
    | LEFT? ANTI
    ;

joinCriteria
    : ON booleanExpression
    | USING identifierList
    ;
```

```scala
  /**
   * Select the proper physical plan for join based on join strategy hints, the availability of
   * equi-join keys and the sizes of joining relations. Below are the existing join strategies,
   * their characteristics and their limitations.
   *
   * - Broadcast hash join (BHJ):
   *     Only supported for equi-joins, while the join keys do not need to be sortable.
   *     Supported for all join types except full outer joins.
   *     BHJ usually performs faster than the other join algorithms when the broadcast side is
   *     small. However, broadcasting tables is a network-intensive operation and it could cause
   *     OOM or perform badly in some cases, especially when the build/broadcast side is big.
   *
   * - Shuffle hash join:
   *     Only supported for equi-joins, while the join keys do not need to be sortable.
   *     Supported for all join types except full outer joins.
   *
   * - Shuffle sort merge join (SMJ):
   *     Only supported for equi-joins and the join keys have to be sortable.
   *     Supported for all join types.
   *
   * - Broadcast nested loop join (BNLJ):
   *     Supports both equi-joins and non-equi-joins.
   *     Supports all the join types, but the implementation is optimized for:
   *       1) broadcasting the left side in a right outer join;
   *       2) broadcasting the right side in a left outer, left semi, left anti or existence join;
   *       3) broadcasting either side in an inner-like join.
   *     For other cases, we need to scan the data multiple times, which can be rather slow.
   *
   * - Shuffle-and-replicate nested loop join (a.k.a. cartesian product join):
   *     Supports both equi-joins and non-equi-joins.
   *     Supports only inner like joins.
   */
  object JoinSelection extends Strategy with PredicateHelper {

    /**
     * Matches a plan whose output should be small enough to be used in broadcast join.
     */
    private def canBroadcast(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
    }

    /**
     * Matches a plan whose single partition should be small enough to build a hash table.
     *
     * Note: this assume that the number of partition is fixed, requires additional work if it's
     * dynamic.
     */
    private def canBuildLocalHashMap(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    }

    /**
     * Returns whether plan a is much smaller (3X) than plan b.
     *
     * The cost to build hash map is higher than sorting, we should only build hash map on a table
     * that is much smaller than other one. Since we does not have the statistic for number of rows,
     * use the size of bytes here as estimation.
     */
    private def muchSmaller(a: LogicalPlan, b: LogicalPlan): Boolean = {
      a.stats.sizeInBytes * 3 <= b.stats.sizeInBytes
    }

    private def canBuildRight(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | LeftOuter | LeftSemi | LeftAnti | _: ExistenceJoin => true
      case _ => false
    }

    private def canBuildLeft(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | RightOuter => true
      case _ => false
    }

    private def getBuildSide(
        wantToBuildLeft: Boolean,
        wantToBuildRight: Boolean,
        left: LogicalPlan,
        right: LogicalPlan): Option[BuildSide] = {
      if (wantToBuildLeft && wantToBuildRight) {
        // returns the smaller side base on its estimated physical size, if we want to build the
        // both sides.
        Some(getSmallerSide(left, right))
      } else if (wantToBuildLeft) {
        Some(BuildLeft)
      } else if (wantToBuildRight) {
        Some(BuildRight)
      } else {
        None
      }
    }

    private def getSmallerSide(left: LogicalPlan, right: LogicalPlan) = {
      if (right.stats.sizeInBytes <= left.stats.sizeInBytes) BuildRight else BuildLeft
    }

    private def hintToBroadcastLeft(hint: JoinHint): Boolean = {
      hint.leftHint.exists(_.strategy.contains(BROADCAST))
    }

    private def hintToBroadcastRight(hint: JoinHint): Boolean = {
      hint.rightHint.exists(_.strategy.contains(BROADCAST))
    }

    private def hintToShuffleHashLeft(hint: JoinHint): Boolean = {
      hint.leftHint.exists(_.strategy.contains(SHUFFLE_HASH))
    }

    private def hintToShuffleHashRight(hint: JoinHint): Boolean = {
      hint.rightHint.exists(_.strategy.contains(SHUFFLE_HASH))
    }

    private def hintToSortMergeJoin(hint: JoinHint): Boolean = {
      hint.leftHint.exists(_.strategy.contains(SHUFFLE_MERGE)) ||
        hint.rightHint.exists(_.strategy.contains(SHUFFLE_MERGE))
    }

    private def hintToShuffleReplicateNL(hint: JoinHint): Boolean = {
      hint.leftHint.exists(_.strategy.contains(SHUFFLE_REPLICATE_NL)) ||
        hint.rightHint.exists(_.strategy.contains(SHUFFLE_REPLICATE_NL))
    }

    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {

      // If it is an equi-join, we first look at the join hints w.r.t. the following order:
      //   1. broadcast hint: pick broadcast hash join if the join type is supported. If both sides
      //      have the broadcast hints, choose the smaller side (based on stats) to broadcast.
      //   2. sort merge hint: pick sort merge join if join keys are sortable.
      //   3. shuffle hash hint: We pick shuffle hash join if the join type is supported. If both
      //      sides have the shuffle hash hints, choose the smaller side (based on stats) as the
      //      build side.
      //   4. shuffle replicate NL hint: pick cartesian product if join type is inner like.
      //
      // If there is no hint or the hints are not applicable, we follow these rules one by one:
      //   1. Pick broadcast hash join if one side is small enough to broadcast, and the join type
      //      is supported. If both sides are small, choose the smaller side (based on stats)
      //      to broadcast.
      //   2. Pick shuffle hash join if one side is small enough to build local hash map, and is
      //      much smaller than the other side, and `spark.sql.join.preferSortMergeJoin` is false.
      //   3. Pick sort merge join if the join keys are sortable.
      //   4. Pick cartesian product if join type is inner like.
      //   5. Pick broadcast nested loop join as the final solution. It may OOM but we don't have
      //      other choice.
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right, hint) =>
        def createBroadcastHashJoin(buildLeft: Boolean, buildRight: Boolean) = {
          val wantToBuildLeft = canBuildLeft(joinType) && buildLeft
          val wantToBuildRight = canBuildRight(joinType) && buildRight
          getBuildSide(wantToBuildLeft, wantToBuildRight, left, right).map { buildSide =>
            Seq(joins.BroadcastHashJoinExec(
              leftKeys,
              rightKeys,
              joinType,
              buildSide,
              condition,
              planLater(left),
              planLater(right)))
          }
        }

        def createShuffleHashJoin(buildLeft: Boolean, buildRight: Boolean) = {
          val wantToBuildLeft = canBuildLeft(joinType) && buildLeft
          val wantToBuildRight = canBuildRight(joinType) && buildRight
          getBuildSide(wantToBuildLeft, wantToBuildRight, left, right).map { buildSide =>
            Seq(joins.ShuffledHashJoinExec(
              leftKeys,
              rightKeys,
              joinType,
              buildSide,
              condition,
              planLater(left),
              planLater(right)))
          }
        }

        def createSortMergeJoin() = {
          if (RowOrdering.isOrderable(leftKeys)) {
            Some(Seq(joins.SortMergeJoinExec(
              leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right))))
          } else {
            None
          }
        }

        def createCartesianProduct() = {
          if (joinType.isInstanceOf[InnerLike]) {
            Some(Seq(joins.CartesianProductExec(planLater(left), planLater(right), condition)))
          } else {
            None
          }
        }

        def createJoinWithoutHint() = {
          createBroadcastHashJoin(canBroadcast(left), canBroadcast(right))
            .orElse {
              if (!conf.preferSortMergeJoin) {
                createShuffleHashJoin(
                  canBuildLocalHashMap(left) && muchSmaller(left, right),
                  canBuildLocalHashMap(right) && muchSmaller(right, left))
              } else {
                None
              }
            }
            .orElse(createSortMergeJoin())
            .orElse(createCartesianProduct())
            .getOrElse {
              // This join could be very slow or OOM
              val buildSide = getSmallerSide(left, right)
              Seq(joins.BroadcastNestedLoopJoinExec(
                planLater(left), planLater(right), buildSide, joinType, condition))
            }
        }

        createBroadcastHashJoin(hintToBroadcastLeft(hint), hintToBroadcastRight(hint))
          .orElse { if (hintToSortMergeJoin(hint)) createSortMergeJoin() else None }
          .orElse(createShuffleHashJoin(hintToShuffleHashLeft(hint), hintToShuffleHashRight(hint)))
          .orElse { if (hintToShuffleReplicateNL(hint)) createCartesianProduct() else None }
          .getOrElse(createJoinWithoutHint())

      // If it is not an equi-join, we first look at the join hints w.r.t. the following order:
      //   1. broadcast hint: pick broadcast nested loop join. If both sides have the broadcast
      //      hints, choose the smaller side (based on stats) to broadcast for inner and full joins,
      //      choose the left side for right join, and choose right side for left join.
      //   2. shuffle replicate NL hint: pick cartesian product if join type is inner like.
      //
      // If there is no hint or the hints are not applicable, we follow these rules one by one:
      //   1. Pick broadcast nested loop join if one side is small enough to broadcast. If only left
      //      side is broadcast-able and it's left join, or only right side is broadcast-able and
      //      it's right join, we skip this rule. If both sides are small, broadcasts the smaller
      //      side for inner and full joins, broadcasts the left side for right join, and broadcasts
      //      right side for left join.
      //   2. Pick cartesian product if join type is inner like.
      //   3. Pick broadcast nested loop join as the final solution. It may OOM but we don't have
      //      other choice. It broadcasts the smaller side for inner and full joins, broadcasts the
      //      left side for right join, and broadcasts right side for left join.
      case logical.Join(left, right, joinType, condition, hint) =>
        val desiredBuildSide = if (joinType.isInstanceOf[InnerLike] || joinType == FullOuter) {
          getSmallerSide(left, right)
        } else {
          // For perf reasons, `BroadcastNestedLoopJoinExec` prefers to broadcast left side if
          // it's a right join, and broadcast right side if it's a left join.
          // TODO: revisit it. If left side is much smaller than the right side, it may be better
          // to broadcast the left side even if it's a left join.
          if (canBuildLeft(joinType)) BuildLeft else BuildRight
        }

        def createBroadcastNLJoin(buildLeft: Boolean, buildRight: Boolean) = {
          val maybeBuildSide = if (buildLeft && buildRight) {
            Some(desiredBuildSide)
          } else if (buildLeft) {
            Some(BuildLeft)
          } else if (buildRight) {
            Some(BuildRight)
          } else {
            None
          }

          maybeBuildSide.map { buildSide =>
            Seq(joins.BroadcastNestedLoopJoinExec(
              planLater(left), planLater(right), buildSide, joinType, condition))
          }
        }

        def createCartesianProduct() = {
          if (joinType.isInstanceOf[InnerLike]) {
            Some(Seq(joins.CartesianProductExec(planLater(left), planLater(right), condition)))
          } else {
            None
          }
        }

        def createJoinWithoutHint() = {
          createBroadcastNLJoin(canBroadcast(left), canBroadcast(right))
            .orElse(createCartesianProduct())
            .getOrElse {
              // This join could be very slow or OOM
              Seq(joins.BroadcastNestedLoopJoinExec(
                planLater(left), planLater(right), desiredBuildSide, joinType, condition))
            }
        }

        createBroadcastNLJoin(hintToBroadcastLeft(hint), hintToBroadcastRight(hint))
          .orElse { if (hintToShuffleReplicateNL(hint)) createCartesianProduct() else None }
          .getOrElse(createJoinWithoutHint())


      // --- Cases where this strategy does not apply ---------------------------------------------
      case _ => Nil
    }
  }
```

- BroadcastHashJoinExec
- BroadcastNestedLoopJoinExec
- ShuffledHashJoinExec
- SortMergeJoinExec

## Reference