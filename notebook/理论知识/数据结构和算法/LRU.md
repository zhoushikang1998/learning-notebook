# 2 LRU

Java代码实现

```java
/*
* LinkedHashMap中的removeEldestEntry会一直返回false；
* 而LinkedHashMap的子类重写该方法后，只要size() > capacity即可实现LRU缓存
*/
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    
    public int get(int key) {
        return super.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        super.put(key, value);
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

