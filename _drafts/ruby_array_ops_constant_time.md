Recently, while doing homeworks for coursera 


Ruby 1.9.2 (p320)
-----------------

Filling up with 'push'

```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.push(i)}; q.shift while q.length>0'
# 0.10 real, 0.06 user, 0.03 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.push(i)}; s.pop while s.length>0'
# 0.08 real, 0.05 user, 0.02 sys
```


Filling up with 'unshift'
```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.unshift(i)}; q.pop while q.length>0'
# 2.53 real, 2.49 user, 0.03 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.unshift(i)}; s.shift while s.length>0'
# 2.48 real, 2.44 user, 0.03 sys
```


Ruby 1.9.3 (p392)
-----------------

Filling up with 'push'

```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.push(i)}; q.shift while q.length>0'
        0.09 real         0.06 user         0.02 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.push(i)}; s.pop while s.length>0'
        0.10 real         0.06 user         0.03 sys
```


Filling up with 'unshift'
```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.unshift(i)}; q.pop while q.length>0'
        2.47 real         2.44 user         0.03 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.unshift(i)}; s.shift while s.length>0'
        2.46 real         2.43 user         0.03 sys
```

Ruby 2.0.0 (p0)
--------------

Filling up with 'push'

```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.push(i)}; q.shift while q.length>0'
        0.11 real         0.07 user         0.03 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.push(i)}; s.pop while s.length>0'
        0.11 real         0.07 user         0.03 sys
```


Filling up with 'unshift'

```
# FIFO
/usr/bin/time ruby -e 'n=100000; q=[]; n.times{|i| q.unshift(i)}; q.pop while q.length>0'
        0.11 real         0.07 user         0.03 sys

# LIFO
/usr/bin/time ruby -e 'n=100000; s=[]; n.times{|i| s.unshift(i)}; s.shift while s.length>0'
        0.11 real         0.07 user         0.02 sys
```
