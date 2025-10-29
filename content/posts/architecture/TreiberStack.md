---
title: "TreiberStack"
date: "2016-04-11"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


```
import javax.annotation.concurrent.ThreadSafe;
import java.util.concurrent.atomic.AtomicReference;

public class TreiberStack {

    @ThreadSafe
    public class ConcurrentStack<E> {
        AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();

        public void push(E item) {
            Node<E> newHead = new Node<E>(item);
            Node<E> oldHead;
            do {
                oldHead = top.get();
                newHead.next = oldHead;
            } while (!top.compareAndSet(oldHead, newHead));
        }

        public E pop() {
            Node<E> oldHead;
            Node<E> newHead;
            do {
                oldHead = top.get();
                if (oldHead == null)
                    return null;
                newHead = oldHead.next;
            } while (!top.compareAndSet(oldHead, newHead));
            return oldHead.item;
        }

        private class Node<E> {
            public final E item;
            public Node<E> next;

            public Node(E item) {
                this.item = item;
            }
        }
    }
}
```