# GraphX

## Graph

- Vertex[VertexRDD]、Edge[EdgeRDD]、Triplet[EdgeTriplet]、RoutingTable

## PartitionStrategy

- EdgePartition2D
- EdgePartition1D
- RandomVertexCut
- CanonicalRandomVertexCut

## Construct

- fromEdgeTuples
- fromEdges
- apply

## Operations

### Property

- numEdges
- numVertices
- inDegrees
- outDegrees
- degrees

### Operator

- collectNeighborIds
- collectNeighbors
- collectEdges
- joinVertices

## Algorithm

- Connected Components
- Label Propagation
- PageRank
- Shortest Paths
- Strongly Connected Components
- SVD++
- Triangle Count

## Reference

- [GraphX Programming Guide](https://spark.apache.org/docs/latest/graphx-programming-guide.html)