
### 这是一个用Netty实现的Redis服务器

### 目的有三:
1. 深入理解redis,将redis上的各种功能慢慢迁移到java上来
	用C语言来实现太麻烦了,而且一般工作中用不到C,正好练习一下Java
	同时降低开发难度,对于一些功能可以更轻松的实现
	未来可以用JNI将重要数据结构封装起来,提高性能(另开一个branch)
2. 理解现代服务器的网络/线程模型
	最近刚刚学习了Netty,顺便熟悉netty的使用
3. 可以用来学习分布式,实现简单的主从,集群功能
   学习分布式不可能只动眼不动手,所以以后可以这个项目为出发点开始魔改

---
### 目前实现的功能:

```
1. get set hget hset
```
---
### Version-1架构
因为使用了Netty,所以不需要过于操心网络,针对redis本身的特点(来往的数据量比较小,全是纯内存操作,其实cpu开销也不是很大),所以单次操作非常的快,这种情况下,cpu并不是瓶颈,往往网络先达到瓶颈(这里不是指网卡流量跑满,而是说处理各种链接的开销)
所以采取单线程(多线程带来的线程同步的开销相当大,暂时没有必要)
也就是 网络 + 业务 都在一个线程里面(一开始也尝试使用额外的线程池来执行,后面发现没有必要)

#### Handler
	用一个Map维护命令和Handler之间的关系,来一个命令就去查找对应的Handler,然后调用对应的handler
		
#### 编码/解码:
	先不考虑性能
	直接使用阿里的FastJson,用来序列化参数(command 对应的类型都利用一个hashMap提前注册好)
    
#### 协议格式	:      
	每个单元 长度 + bytes
	一共三个单元:
	requestId(用来校验)
	command(命令类型)
	content(结果,也是FastJson序列化的字节)
	

#### 数据库部分：
	在RedisDb.java文件里面,核心是一个 
	HashMap<String,RedisObject>
	
RedisObject定义如下:

```
class RedisObject{
	int type;
	Object data;
	.....
}	
```

####  实现了HyperLogLog数据结构：
##### 基本原理:
 
 *  令K 为n次投掷「硬币」面最大的投掷次数
 *  P(A) = 所有投掷次数<a href="https://www.codecogs.com/eqnedit.php?latex=$&space;P(X_i&space;<=&space;K)&space;=&space;(1&space;-&space;(1/2)^k)^n$" target="_blank"><img src="https://latex.codecogs.com/gif.latex?$&space;P(X_i&space;<=&space;K)&space;=&space;(1&space;-&space;(1/2)^k)^n$" title="$ P(X_i <= K) = (1 - (1/2)^k)^n$" /></a>
 *  P(B) = 不是所有投掷次数X_i 都小于K(因为至少有一个达到了k) = <a href="https://www.codecogs.com/eqnedit.php?latex=\bg_black&space;\large&space;$&space;1&space;-&space;(1&space;-&space;(1/2)^{k-1})^n&space;$" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\bg_black&space;\large&space;$&space;1&space;-&space;(1&space;-&space;(1/2)^{k-1})^n&space;$" title="\large $ 1 - (1 - (1/2)^{k-1})^n $" /></a>
 *  注意 A,B 是同时发生的,因此P(A)和P(B)就不应该接近0,否则和「小概率事件不发生的假设」矛盾
 *  容易得到 n  约等于<a href="https://www.codecogs.com/eqnedit.php?latex=\large&space;$&space;2^K&space;$" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\large&space;$&space;2^K&space;$" title="\large $ 2^K $" /></a>

 * 分桶进行统计:
 *      为了避免出现只投了一次就出现很多正面的情况,这会造成偶然误差很大,所以通过分桶来减小误差
 *      随着实验次数的增多,这里的误差就会减小,但是如果只实验几次,即便分桶也无济于事,很可能造成误差很大
 *      比如一共就只实验10次,完全有可能造成里面有一半的桶误差都很大的情况
 *     实际上Redis本身做了很多优化,特别是利用一个关于没有投中桶的次数$ez$的函数进行拟合

#####  实现步骤:
1. 对字符串进行hash,这里使用`MD5`进行hash,然后获取前64位作为一个long用来进行标志
2. 取后14位作为桶的索引,取前50位作为投掷的序列,这里曾经犯了一个错误:
		

```
. 利用 H 的后14位作为index,然后计算整个H的zeroBits,注意这个时候没有使用Hash的后50位,从而导致bug:
  注意到: 由于bits大于14的概率很小,
  所以大多数时候实验的结果都是由后面14位决定的,
  这就导致同一个index上面的,bit几乎都是一样的
  完全失去了随机的意义	
```

3. 计算低位连续为0的个数(这里是对投掷次数进行建模),计算方式如下:
	* 获取一个整数末尾有多少连续的0
	*  1 计算 `x = (val ^ (val - 1)) & val`
	*  2 `x = 2 ^ i`, i 就是末尾连续是0的个数,i最多等于50
	*  3 查表得 `i = table.get(x)`(可能还有其它的方法,比如二分移位进行试探,由于这里x**绝大多数**情况下都是0或者1,所以可以从1开始试探)
	*  4 将 `i + 1`和原来桶里面的值取大者保留, 这是为了计算1出现的位置(实际上等于至少投掷一次)
	*  5 算count的时候计算`调和平均数`,这是为了避免单个离散值导致的异常

##### 实现:

        // 计算估计的基数,利用调和平均数进行计算
    public long estimateForHLL(){
        double sumbitsInverse = 0;
        int validNum = 0;// 统计不为0的桶
        for(int i = 0; i < Num; i++){
            int bucketEstimate = buckets[i];// 连续0的数量+1,第一次出现1的index
            if(bucketEstimate != 0){
                sumbitsInverse += reciprocal[bucketEstimate];// 2 ^ -i
                validNum++;
            }
        }
        if(validNum < Num * 0.05) {
            // 如果数量很少就直接计数
            return validNum;
        }
        
        int ez = Num - validNum;// 0的数量
        sumbitsInverse += ez;// 当ez很大的时候进行的补偿(具体推导不明)
        double zl = Math.log(ez + 1);
        double beta = -0.370393911 * ez +
                0.070471823 * zl +
                0.17393686 * Math.pow(zl,2) +
                0.16339839 * Math.pow(zl,3) +
                -0.09237745 * Math.pow(zl,4) +
                0.03738027 * Math.pow(zl,5) +
                -0.005384159 * Math.pow(zl,6) +
                0.00042419 * Math.pow(zl,7);


        double alpha =  0.7213 / (1 + 1.079 / Num );
        double ret = (alpha * Num * validNum * (1 / (sumbitsInverse + beta)));
        return (long)ret;
    }
##### 实际上这几个参数一个都不能少,否则误差就会变大(原理和实现还是差距很大的,具体推导感兴趣的自己取看paper,redis源码里面给了paper的名字)