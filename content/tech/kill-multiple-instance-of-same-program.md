---
author: "Nishant M Gandhi"
date: 2017-11-08
linktitle: Kill Multiple Instance of same program in Linux
title: Kill Multiple Instance of same program in Linux
image : ""
---

When you have multiple instance of same program running in background, <br/>
and you want to kill them all, <br/>
Use the below command after replace **processName** with your program name.


    ps -A | grep processName | awk '{print $1}' | xargs -L1 kill -9

Basics, the pipe symbol "|", which is used as connector between two commands, where output of first command is passed as input for second program.

Let us breakdown each command passed though multiple pipes.

**Step 1:** Get list of all processes running in the system in rows. <br/>

    ps -A

**Step 2:** Filter the process rows we want to kill. <br/>
    
    grep processName

**Step 3:** Pick the first-column-data (process ID) of filtered process rows.

    awk '{print $1}'

**Step 4:** Kill all filtered processes using there process-ID.

    xargs -L1 kill -9
