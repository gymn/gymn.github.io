---
title: 你好，初见
categories:
 -Hello
---
你好，欢迎光临古月慕南的个人博客,以后我会在这里记录自己的技术成长，欢迎多多指教哦。
```java
 public boolean hasCycle(ListNode head) {
    if(head==null){
        return false;
    }
    ListNode slow = head;
    ListNode fast = head;
    do{
        slow = slow.next;
        if(fast.next==null||fast.next.next==null){
            return false;
        }
        fast = fast.next.next;
    } while(!slow.equals(fast));
    return true;
}
```
![hi](/assets/images/hi.jpeg "你好")