### 1. 在g++ 中用一个option 来指定

```
g++ -o main main.cpp -I /usr/local/include/python/
```

### 2. 通过环境变量来设置，这样就可以不要在g++ 中来指定了

```
export CPLUS_INCLUDE_PATH=/usr/local/include/python/
```

g++ -o main main.cpp