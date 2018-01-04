---
title: "Implementing HashTable in Java"
date: 2016-04-08T22:28:00-08:00
draft: false
---
I have been asked before about how would I implement Hashtable in Java.

Before going into code, here are the steps I have used to implement HashTable.

- Encapsulate key, value pair and linked list  with `HashEntry` class.  
- Create buckets (Just an array of `HashEntry`)
- `put(key, value)` calculates hash from `key` and inserts in bucket at position index=hash.  
  * Incase we already have an entry for the hash, insert entry at the end of linked list maintained by entry.  
- `get(key)` calculates hash, and returns the entry from bucket derived from hash.  
  * If bucket contains multiple elements for same hash, find the entry in linked list of the bucket and return value.  
  * If the element is not found, return `null`.  

This is not meant to be complete solution. Just bare bones.

{{< gist amadamala 3cdd53cb5a6b1c1df540981ab0245479 >}}

Here is the console output from running the above code.

{{< highlight java >}}
****   HashTable  ***

 bucket[0] = [0, 0] -> [11, 11] -> [22, 22]
 bucket[1] = [1, 1] -> [12, 12] -> [23, 23]
 bucket[2] = [2, 2] -> [13, 13] -> [24, 24]
 bucket[3] = [3, 3] -> [14, 14] -> [25, 25]
 bucket[4] = [4, 4] -> [15, 15] -> [26, 26]
 bucket[5] = [5, 5] -> [16, 16] -> [27, 27]
 bucket[6] = [6, 6] -> [17, 17] -> [28, 28]
 bucket[7] = [7, 7] -> [18, 18] -> [29, 29]
 bucket[8] = [8, 8] -> [19, 19]
 bucket[9] = [9, 9]
 bucket[10] = [20, 20]
 bucket[11] = [10, 10] -> [21, 21]

Value for key(20) : 20
{{< / highlight >}}