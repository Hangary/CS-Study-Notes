# Network Structure

Two Important Network Properties:
1. **Clustering Coefficient** (CC): Probility(A-B has an edge, given an A-C edge and a C-B edge)
2. **Path Length** of shortest path

Different kinds of networks:
- **Extended Ring Graph**: high CC, long paths
- **Random Graph**: low CC, short paths
- **Small World Networks**: high CC, short paths
  - Most "natural evolved" networks are small world:
    - Network of actors => six degrees of Kevin Bacon
    - Network of humans => Milgram's experiment
    - WWW, the Internet

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203201357.png)

**Emergent phenomena**: simple end behavior => leads to => complex system-wide behavior

**Degree** of a vertex is the number of its immediate neighbor vertices.
- **Degree distribution**: what is the probability of a given node having $k$ edges (neighbors, friends, …)