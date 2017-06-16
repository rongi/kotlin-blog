---
layout: post
title:  "Java vs Kotlin: Count number of items in a list satisfying a condition"
categories: kotlin
---

Java:

```java
  void myFunction {
    int selectedItemsCount = countSelectedItems(cells);
  }

  private static int countSelectedItems(List<Cell> cells) {
    int count = 0;

    for (Cell cell : cells) {
      if (cell.isSelected()) {
        ++count;
      }
    }

    return count;
  }
```

Kotlin:

```kotlin
fun myFunction {
  val selectedItemsCount = cells.count { it.selected }
}
```

Cathartic
