𝐆𝐨𝐥𝐝𝐞𝐧 𝐑𝐮𝐥𝐞𝐬 𝐭𝐨 𝐚𝐧𝐬𝐰𝐞𝐫 𝐢𝐧 𝐚 𝐒𝐲𝐬𝐭𝐞𝐦 𝐃𝐞𝐬𝐢𝐠𝐧 𝐈𝐧𝐭𝐞𝐫𝐯𝐢𝐞𝐰
Sharing with you 30 golden rules to answer in System Design Interviews. The rules are as follows -

| Scenario / Requirement | Recommended Solution / Concept |
| :--- | :--- |
| **Read-heavy system** | Cache |
| **Low latency** | Cache & CDN |
| **Write-heavy system** | Message Queue (Async processing) |
| **ACID compliance** | RDBMS / SQL Database |
| **Unstructured data (No ACID)** | No-SQL Database |
| **Complex data (videos, images, files)** | Blob / Object Storage |
| **Complex/heavy pre-computation** | Message Queue & Cache |
| **High volume data search** | Search Index, Tries, or Elasticsearch |
| **Scale SQL Database** | Database Sharding |
| **High Availability, Performance, Throughput** | Load Balancer |
| **Global data delivery (Reliability, HA)** | CDN |
| **Graph data (nodes, edges, relationships)** | Graph Database |
| **Scaling components (servers, DBs)** | Horizontal Scaling |
| **High-performing DB queries** | Database Indexes |
| **Bulk job processing** | Batch Processing & Message Queues |
| **Reduce server load / DOS protection** | Rate Limiter |
| **Microservices** | API Gateway (Auth, SSL, Routing) |
| **Single point of failure** | Redundancy |
| **Fault-tolerance & durability** | Data Replication |
| **Bi-directional user communication** | Websockets |
| **Failure detection in distributed systems** | Heartbeat |
| **Data integrity** | Checksum Algorithm |
| **Efficient scaling (no hotspots)** | Consistent Hashing |
| **Decentralized data transfer** | Gossip Protocol |
| **Location data (maps, nearby)** | Quadtree, Geohash |
| **_Naming Convention_** | Use generic names (e.g., Message Queue) vs specific (e.g., Kafka) |
| **High Availability (Consistency)** | Eventual Consistency (Strong consistency often not possible) |
| **Domain name resolution** | DNS (Domain Name System) |
| **Limiting large response data** | Pagination |
| **Cache eviction policy** | LRU (Least Recently Used) Cache |