# AOP
## 同一个方法上有多个切面，各个切面的执行顺序怎样控制
```text
通过@Order注解来控制，数字越小，越先执行，先执行的最后结束。
```
## execution表达式（切面表达式）内容
```text
修饰符 返回值 包名 方法名（参数） 异常
public * com.example.service.*.*(..)
```