# cs-maps-hashing-readme

## Learning goals 

1.  Analyze the performance of a simple `Map` implementation.
2.  Implement the `Map` interface using multiple lists.
3.  Implement a hash table.
4.  Write a hash function for String-like objects.


## Overview

In this readme, we'll look at solutions to the previous lab and analyze the performance of `MyLinearMap`.  Then we'll define `MyBetterMap`, an implementation of the `Map` interface that uses many instances of `MyLinearMap`; and we'll introduce **hashing**, which makes `MyBetterMap` more efficient.  Finally, we'll demonstrate a function that implements hashing for a String-like object.


## Analyzing `MyLinearMap`

We'll start with our solutions to the previous lab and then analyze their performance.  Here are `findEntry` and `equals`:

```java
private Entry findEntry(Object target) {
	for (Entry entry: entries) {
		if (equals(target, entry.getKey())) {
			return entry;
		}
	}
	return null;
}

private boolean equals(Object target, Object obj) {
	if (target == null) {
		return obj == null;
	}
	return target.equals(obj);
}
```

The runtime of `equals` might depend on the size of the `target` and the `keys`, but does not generally depend on the number of entries, `n`.  So `equals` is constant time.

In `findEntry`, we might get lucky and find the key we're looking for at the beginning, but we can't count on it.  In general, the number of entries we have to search is proportional to `n`, so `findEntry` is linear.

Most of the core methods in `MyLinearMap` use `findEntry`, including `put`, `get`, and `remove`.  Here's what they look like:

```java
public V put(K key, V value) {
	Entry entry = findEntry(key);
	if (entry == null) {
		entries.add(new Entry(key, value));
		return null;
	} else {
		V oldValue = entry.getValue();
		entry.setValue(value);
		return oldValue;
	}
}

public V get(Object key) {
	Entry entry = findEntry(key);
	if (entry == null) {
		return null;
	}
	return entry.getValue();
}
	
public V remove(Object key) {
	Entry entry = findEntry(key);
	if (entry == null) {
		return null;
	} else {
		V value = entry.getValue();
		entries.remove(entry);
		return value;
	}
}
```

After `put` calls `findEntry`, everything else is constant time.  Remember that `entries` is an `ArrayList`, so adding an element *at the end* is constant time, on average.  If the key is already in the map, we don't have to add an entry, but we have to call `entry.getValue` and `entry.setValue`, and those are both constant time.  Adding it all up, `put` is linear.

By the same reasoning, `get` is also linear.

`remove` is slightly more complicated because `entries.remove` might have to remove an element from the beginning or middle of the `ArrayList`, and that takes linear time.  But that's ok: two linear operations are still linear.

In summary, the core methods are all linear, which is why we called this implementation `MyLinearMap` (ta-da!).

If we know that the number of entries will be small, this implementation might be good enough, but we can do better.  In fact, there is an implementation of `Map` where all of the core methods are constant time.  When you first hear that, it might not seem possible.  What we are saying, in effect, is that you can find a needle in a haystack in constant time, regardless of how big the haystack is.  It's magic.

We'll explain how it works in two steps:

1.  Instead of storing entries in one big `List`, we'll break them up into lots of short lists.  For each key, we'll use a **hash code** (explained in the next section) to determine which list to use.

2.  Using lots of short lists is faster than using just one, but as we'll explain, it doesn't change the order of growth; the core operations are still linear.  But there is one more trick: if we increase the number of lists to limit the number of entries per list, the result is a constant-time map.  You'll see the details in the next lab, but first: hashing!


## Hashing

To improve the performance of `MyLinearMap`, we'll define a new class, called `MyBetterMap`, that contains a collection of `MyLinearMap` objects.  It divides the keys among the embedded maps, so the number of entries in each map is smaller, which speeds up `findEntry` and the methods that depend on it.

Here's the beginning of the class definition:

```java
public class MyBetterMap<K, V> implements Map<K, V> {
	
	protected List<MyLinearMap<K, V>> maps;
	
	public MyBetterMap(int k) {
		makeMaps(k);
	}

	protected void makeMaps(int k) {
		maps = new ArrayList<MyLinearMap<K, V>>(k);
		for (int i=0; i<k; i++) {
			maps.add(new MyLinearMap<K, V>());
		}
	}
}
```

The instance variable, `maps` is a collection of `MyLinearMap` objects.  The constructor takes a parameter, `k`, that determines how many maps to use, at least initially.  Then `makeMaps` creates the embedded maps and stores them in an `ArrayList`.

Now, the key to making this work is that we need some way to look at a key and decide which of the embedded maps it should go into.  When we `put` a new key, we choose one of the maps; when we `get` the same key, we have to remember where we put it.

One possibility is to choose one of the sub-maps at random and keep track of where we put each key.  But how should we keep track?  It might seem like we could use a `Map` to look up the key and find the right sub-map, but the whole point of this exercise is to write an efficient implementation of a `Map`.  We can't assume we already have one.

A better approach is to use a **hash function**, which takes an Object, any Object, and returns an integer called a **hash code**.  Importantly, if it sees the same Object more than once, it always returns the same hash code.  That way, if we use the hash code to store a key, we'll get the same hash code when we look it up.

In Java, every `Object` provides a method called `hashCode` that computes a hash function.  The implementation of this method is different for different objects; we'll see an example soon.

Here's the helper function we wrote to choose the right sub-map for a given key:

```java
protected MyLinearMap<K, V> chooseMap(Object key) {
	int index = key==null ? 0 : key.hashCode() % maps.size();
	return maps.get(index);
}
```

If `key` is `null`, we choose the sub-map with index 0, arbitrarily.  Otherwise we use `hashCode` to get an integer and then apply the modulus operator, `%`, which guarantees that the result is between 0 and `maps.size()-1`.  So `index` is always a valid index into `maps`.  Then `chooseMap` returns a reference to the map it chose.

We use `chooseMap` in both `put` and `get`, so when we look up a key, we get the same map we chose when we added the key.  At least, we should — we'll explain a little later why this might not work.

Here's our implementation of `put` and `get`:

```java
public V put(K key, V value) {
  MyLinearMap<K, V> map = chooseMap(key);
	return map.put(key, value);
}

public V get(Object key) {
	MyLinearMap<K, V> map = chooseMap(key);
	return map.get(key);
}
```

Pretty simple, right?  In both methods, we use `chooseMap` to find the right sub-map and then invoke a method on the sub-map.  In the next lab, you'll have a chance to finish off a few other methods.  But first, let's think about performance.

If there are `n` entries split up among `k` sub-maps, there will be `n/k` entries per map, on average.  When we look up a key, we have to compute its hash code, which takes some time, then we search the corresponding sub-map.  Because the entry lists in `MyBetterMap` are `k` times shorter than the entry list in `MyLinearMap`, we expect the search to be `k` times faster.  But the run time is still proportional to `n`, so `MyBetterMap` is still linear.  In the next lab, you'll see how we can fix that.

But first, a little more about hashing.


## How does hashing work?

The fundamental requirement for a hash function is that the same object should produce the same hash code every time.  For immutable objects, that's relatively easy.  For objects with mutable state, we have to think harder.

As an example of an immutable object, we'll define a class called `SillyString` that encapsulates a String:

```java
public class SillyString {
	private final String innerString;

	public SillyString(String innerString) {
		this.innerString = innerString;
	}

	public String toString() {
		return innerString;
	}
```

This class is not very useful, which is why it's called `SillyString`, but we'll use it to show how a class can define its own hash function:

```java
@Override
public boolean equals(Object other) {
	return this.toString().equals(other.toString());
}
	
@Override
public int hashCode() {
	int total = 0;
	for (int i=0; i<innerString.length(); i++) {
		total += innerString.charAt(i);
	}
	System.out.println(total);
	return total;
}
```

Notice that `SillyString` overrides both `equals` and `hashCode`.  This is important.  In order to work properly, `equals` has to be consistent with `hashCode`, which means that if two objects are considered equal — that is, `equals` returns `true` — they should have the same hash code.  But this requirement only works one way; if two objects have the same hash code, they don't necessarily have to be equal.

`equals` works by invoking `toString`, which returns `innerString`.  So two `SillyString` objects are equal if their `innerString` attributes are equal.

`hashCode` works by iterating through the characters in the String and adding them up.  When you add a character to an `int`, Java converts the character to an integer using its Unicode code point.  You don't need to know anything about Unicode to understand this example, but if you are curious, [you can read more here](https://en.wikipedia.org/wiki/Code_point).

This hash function satisfies the requirement: if two `SillyString` objects contain embedded strings that are equal, they will get the same hash code.

This works correctly, but the performance might not be very good, because it returns the same hash code for many different strings.  If two strings contain the same letters in any order, they will have the same hash code.  And even if they don't contain the same letters, they might yield the same total, like `"ac"` and `"bb"`.

If many objects have the same hash code, they end up in the same sub-map, and some sub-maps have more entries than others.  In that case, the speedup when we have `k` maps might be much less than `k`.  So one of the goals of a hash function is to be uniform; that is, it should be equally likely to produce any value in the range.  You can [read more about designing good hash functions here](https://en.wikipedia.org/wiki/Hash_function).


## Hashing and mutation

Strings are immutable and `SillyString` is also immutable because `innerString` is declared to be `final`.  Once you create a `SillyString`, you can't make `innerString` refer to a different String, and you can change the String it refers to.  Therefore, it will always have the same hash code.

But let's see what happens with a mutable object.  Here's a definition for `SillyArray`, which is identical to `SillyString`, except that it uses an array of characters instead of a String:

```java
public class SillyArray {
	private final char[] array;

	public SillyArray(char[] array) {
		this.array = array;
	}

	public String toString() {
		return Arrays.toString(array);
	}
	
	@Override
	public boolean equals(Object other) {
		return this.toString().equals(other.toString());
	}
	
	@Override
	public int hashCode() {
		int total = 0;
		for (int i=0; i<array.length; i++) {
			total += array[i];
		}
		System.out.println(total);
		return total;
	}
```

`SillyArray` also provides `setChar`, which makes it possible to modify the characters in the array:

```java
public void setChar(int i, char c) {
	this.array[i] = c;
}
```

Now suppose we create a `SillyArray` and add it to a map:

```java
SillyArray array1 = new SillyArray("Word1".toCharArray());
map.put(array1, 1);
```

The hash code for this array is 461.  Now if we modify the contents of the array and they try to look it up:

```java
array1.setChar(0, 'C');
Integer value = map.get(array1);
```

The hash code after the mutation is 441.  With a different hash code, there's a good chance we'll go looking in the wrong sub-map.  In that case, we won't find the key, even though it is in the map.  And that's bad.

In general, it is dangerous to use mutable objects as keys in data structures that use hashing, which includes `MyBetterMap` and `HashMap`.  If you can guarantee that the keys won't be modified while they are in the map, or that any changes won't affect the hash code, it might be okay.  But it is probably a good idea to avoid it.


## Resources

[Unicode code point](https://en.wikipedia.org/wiki/Code_point): Wikipedia.

[Hash function](https://en.wikipedia.org/wiki/Hash_function): Wikipedia.
