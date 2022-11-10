

## 临时使用

```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple some-package
```





## 永久



```bash
python -m pip install --upgrade pip
```



```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



## 取消

```bash
pip config unset global.index-url
```





## 查看现在用的



```bash
pip config list
```



> global.index-url='https://pypi.tuna.tsinghua.edu.cn/simple'





