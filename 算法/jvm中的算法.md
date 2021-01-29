```c++
static int binary_search(Array<Method*>* methods, Symbol* name) {
  int len = methods->length();
  // methods are sorted, so do binary search
  int l = 0;
  int h = len - 1;
  while (l <= h) {
    int mid = (l + h) >> 1;
    Method* m = methods->at(mid);
    assert(m->is_method(), "must be method");
    int res = m->name()->fast_compare(name);
    if (res == 0) {
      return mid;
    } else if (res < 0) {
      l = mid + 1;
    } else {
      h = mid - 1;
    }
  }
  return -1;
}
```

```c++
static int linear_search(Array<Method*>* methods, Symbol* name, Symbol* signature) {
  int len = methods->length();
  for (int index = 0; index < len; index++) {
    Method* m = methods->at(index);
    assert(m->is_method(), "must be method");
    if (m->signature() == signature && m->name() == name) {
       return index;
    }
  }
  return -1;
}
```

