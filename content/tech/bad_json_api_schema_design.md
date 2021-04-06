---
author: "Nishant M Gandhi"
date: 2021-04-06
linktitle: Some Bad JSON Schema Design to avoid for API
title: Some Bad JSON Schema Design to avoid for API
image : ""
---

### **Don't use dynamic keys**

The keys in JSON schema should be fixed and documentable. Making keys dynamic  in JSON schema, will make users left to guess the keys to retrieve data. It is also hard to document dynamic schema and create huge problem with schema versioning.
Example of Dynamic Keys.

``` python
[{'1': XYZ}, {'20': PQR}, {'31': ABC}]
```
