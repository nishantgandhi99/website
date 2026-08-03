---
author: "Nishant M Gandhi"
date: 2026-07-03
linktitle: My Thoughts on HubSpot's AI Infrastructure Blog
title: My Thoughts on HubSpot's AI Infrastructure Blog
image : ""
---

I was reading HubSpot’s engineering blog on building the AI retrieval infrastructure behind 20+ billion vectors and it was a good reminder of how quickly retrieval systems become infrastructure problems.
When we talk about RAG or semantic search, the discussion often starts with embeddings, vector databases and model quality. But at this scale, the hard part is not just storing vectors or running nearest neighbor search. The hard part is keeping the system reliable, observable and easy to operate in production.

What stood out to me was the evolution of their architecture:
+ Moving away from manual and error-prone Qdrant deployments
+ Building automation around Kubernetes operators
+ Treating vector search as a production platform, not just an application dependency
+ Designing for scale, reliability and day-to-day operations from the beginning

It is easy to underestimate how much operational complexity sits behind a good retrieval layer. Index lifecycle management, cluster provisioning, upgrades, backups, scaling, monitoring, failure recovery. None of these are as exciting as talking about LLMs, but they are usually what decide whether the system works reliably.

The scale of 20+ billion vectors is impressive, but the more useful takeaway for me was the mindset. Retrieval infrastructure needs to be designed like any other critical distributed system. Automation, reliability and debuggability matter just as much as retrieval quality.
Definitely worth a read if you are working on RAG systems, semantic search or production AI infrastructure.

Blog Link: https://lnkd.in/gW8Ga8nR