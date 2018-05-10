---
author: "Nishant M Gandhi"
date: 2018-05-09
linktitle: How to append csv file multiple times
title: How to append csv file multiple times
image : ""
---
# How to append csv file multiple times while skipping the header using shell command

There are two problem here. First is to append file multiple times and Second is to skip the header of csv. 

For the first part, we will use '**for-loop**' and append operator '**>>**' to append file n times.

For the second part, we will use '**tail**' command with argument '**-n +2**'. The '**+2**' help us to skip first line of a file and only print file starting from 2nd line. 

Let's take an example where you want to append same file with header 1000 times to create bigger file.

we will run this command to create file that skips header from original file and create mybigfile. 

	for i in {1..999};do tail -n +2 original.csv >> mybigfile.csv; done

Now, we only ran the above loop for 999 time because we still need header in mybigfile. We will run the last command as below to append original file with header for our desination file to also have header.

	cp original.csv mybigfile_with_header.csv
	cat mybigfile.csv >> mybigfile_with_header.csv

I acknowledge that the methos above may not be the most efficient way to do it but it gets job done.