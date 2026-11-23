---
title: "Implementing HashTable in Java"
date: 2016-04-08T22:28:00-08:00
draft: false
---
I have been asked before how I would implement a hash table in Java.

Before going into the code, here are the steps I used to implement `HashTable`.

- Encapsulate the key, the value, and a link to the next entry in a `HashEntry` class.
- Create the buckets, which are an array of `HashEntry`.
- `put(key, value)` calculates the bucket index from the hash code of `key`.
  * If the bucket is empty, the new entry becomes the first entry of the bucket.
  * If the bucket already has entries, walk the linked list of the bucket. If an entry has the same key, replace its value. Otherwise, add the new entry at the end of the linked list.
- `get(key)` calculates the bucket index, and then walks the linked list of the bucket.
  * If an entry has the same key, return its value.
  * If no entry has the same key, return `null`.

The code is not meant to be a complete solution, and it only has the bare bones. For example, the number of buckets is fixed at 16, so the linked lists get longer as you add more keys. A complete hash table creates a larger array and moves the entries when the table fills up, and it also supports removing a key.

{{< highlight java >}}
public class HashTable {
    private static final int BUCKET_COUNT = 16;
    private final HashEntry[] buckets = new HashEntry[BUCKET_COUNT];

    /**
     * Adds the key and value. If the key is already in
     * the table, replaces its value.
     */
    public void put(String key, String value) {
        int index = getIndex(key);
        HashEntry entry = buckets[index];
        if (entry == null) {
            buckets[index] = new HashEntry(key, value);
            return;
        }
        // Walk the linked list of the bucket. Update the
        // value if the key exists, or else add a new
        // entry at the end of the list.
        while (true) {
            if (entry.key.equals(key)) {
                entry.value = value;
                return;
            }
            if (entry.next == null) {
                entry.next = new HashEntry(key, value);
                return;
            }
            entry = entry.next;
        }
    }

    /**
     * Returns 'null' if the key is not found.
     */
    public String get(String key) {
        HashEntry entry = buckets[getIndex(key)];
        // Check the linked list of the bucket for a
        // match for the given 'key'.
        while (entry != null) {
            if (entry.key.equals(key)) {
                return entry.value;
            }
            entry = entry.next;
        }
        return null;
    }

    private int getIndex(String key) {
        if (key == null) {
            throw new IllegalArgumentException("key is null");
        }
        // Piggy backing on the java string hashcode
        // implementation. hashCode() can be negative, and
        // floorMod keeps the index between 0 and
        // BUCKET_COUNT - 1.
        return Math.floorMod(key.hashCode(), BUCKET_COUNT);
    }

    private static class HashEntry {
        final String key;
        String value;
        // Next entry in the same bucket.
        HashEntry next;

        HashEntry(String key, String value) {
            this.key = key;
            this.value = value;
        }

        @Override
        public String toString() {
            return "[" + key + ", " + value + "]";
        }
    }

    @Override
    public String toString() {
        StringBuilder hashTableStr = new StringBuilder();
        for (int i = 0; i < buckets.length; i++) {
            HashEntry entry = buckets[i];
            if (entry == null) {
                continue;
            }
            hashTableStr.append("\n bucket[")
                    .append(i)
                    .append("] = ")
                    .append(entry);
            entry = entry.next;
            while (entry != null) {
                hashTableStr.append(" -> ").append(entry);
                entry = entry.next;
            }
        }
        return hashTableStr.toString();
    }

    public static void main(String[] args) {
        HashTable hashTable = new HashTable();
        // Put some key values.
        for (int i = 0; i < 30; i++) {
            final String key = String.valueOf(i);
            hashTable.put(key, key);
        }

        // Print the HashTable structure
        log("****   HashTable  ***");
        log(hashTable.toString());
        log("\nValue for key(20) : " + hashTable.get("20"));

        // Putting an existing key replaces its value.
        hashTable.put("20", "twenty");
        log("Value for key(20) after second put : " + hashTable.get("20"));

        // A key that is not in the table returns null.
        log("Value for key(99) : " + hashTable.get("99"));

        // "polygenelubricants".hashCode() is negative.
        hashTable.put("polygenelubricants", "negative hash");
        log("Value for key(polygenelubricants) : "
                + hashTable.get("polygenelubricants"));
    }

    private static void log(String msg) {
        System.out.println(msg);
    }
}
{{< / highlight >}}

Here is the console output from running the above code.

{{< highlight text >}}
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
 bucket[14] = [20, 20]
 bucket[15] = [10, 10] -> [21, 21]

Value for key(20) : 20
Value for key(20) after second put : twenty
Value for key(99) : null
Value for key(polygenelubricants) : negative hash
{{< / highlight >}}

### Update, November 2026

I fixed the following problems in the first version of the code:

- `get(key)` returned the value of the last entry in the bucket when the key was not in the table. It now returns `null`.
- `put(key, value)` added a second entry when the key was already in the table. It now replaces the value.
- The bucket index was `key.hashCode() % 16`, which is negative when the hash code is negative, so `put` threw an `ArrayIndexOutOfBoundsException` for those keys. The code now uses `Math.floorMod`.
- `toString()` numbered only the buckets that had entries, so the printed bucket numbers were wrong after the first empty bucket. It now prints the actual bucket index.
