## tutorial 07 notes ##

# bug #

语句：

~~~c
c->top -= 32 - sprintf(lept_context_push(c, 32), "%.17g", v->u.n);
~~~

执行前c->top=0;
执行后c->top=18xxxxxxxxx(一个很大的值）;该值为2^64左右，判断其是以0 - 1 所得，c->top提前压入栈，而非执行lept_context_push后的值。
改为冗长的写法后正常工作，sprintf返回值为1。gcc版本为4.8.4,猜测与编译器有很大关系

# 生成字符串 #
主要问题是：如何将解码UTF-8字符串，将其转换成原来的ASCII字符串。
思路：
1. 先将解码出utf-8码点中对应的值。
2. 再转换成ASCII字符串

下表是utf-8码点如何编码成一个个字节存储的。

| 码点范围            | 码点位数  | 字节1     | 字节2    | 字节3    | 字节4     |
|:------------------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| U+0000 ~ U+007F    | 7        | 0xxxxxxx |
| U+0080 ~ U+07FF    | 11       | 110xxxxx | 10xxxxxx |
| U+0800 ~ U+FFFF    | 16       | 1110xxxx | 10xxxxxx | 10xxxxxx |
| U+10000 ~ U+10FFFF | 21       | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |

从第一个字节的前几位便可判断其为何种码点，若最高位为0，那必为U+0000~U+007F范围内的码点，表中的`x`表示码点上的一位数据，以U+0000~U+007F范围中的码点为例，字节1的低7位组成了其全部的码点数据。

解码成对应码点的代码如下：

~~~c
        if((s[i] >> 7) == 0x00){
          u = s[i];
          .....
        }
        else if((s[i] >> 5) == 0x06){
            u = s[i] & 0x1f;
            u = (u << 5) | (s[i+1]& 0x3f);                    
        }
         else if((s[i] >> 4) == 0x0e){
             u = s[i] & 0x0f;
             u = (u << 4) | (s[i+1] & 0x3f);
             u = (u << 6) | (s[i+2] & 0x3f);
             i +=2;
             ....
         }
         else if((s[i] >> 3) == 0x1e){
             u = s[i] & 0x07;
             u = (u << 3) | (s[i+1] & 0x3f);
             u = (u << 6) | (s[i+2] & 0x3f);
             u = (u << 6) | (s[i+3] & 0x3f);
             i += 3;
             ....
         }
~~~

接下来就是将其转换成对应的ASCII字符串，那么首先要清楚一个json的字符串是如何被转换成utf-8的码点的。

~~~
string = quotation-mark *char quotation-mark
char = unescaped /
   escape (
       %x22 /          ; "    quotation mark  U+0022
       %x5C /          ; \    reverse solidus U+005C
       %x2F /          ; /    solidus         U+002F
       %x62 /          ; b    backspace       U+0008
       %x66 /          ; f    form feed       U+000C
       %x6E /          ; n    line feed       U+000A
       %x72 /          ; r    carriage return U+000D
       %x74 /          ; t    tab             U+0009
       %x75 4HEXDIG )  ; uXXXX                U+XXXX
escape = %x5C          ; \
quotation-mark = %x22  ; "
unescaped = %x20-21 / %x23-5B / %x5D-10FFFF
~~~

一个json的字符串以`"`开始和结束，普通的字符直接编码，而遇到以`\`开始的会进行转义，只需进行对应的转换即可。而\uXXXX的是一些特殊UTF-8的码点, 如0x20以下的部分码点，需要按\u00xx的格式输出。注意部分的转义字符也小于0x20，因此需先判断是不是转义字符，若不是再判断是不是小于0x20.

~~~c
if(!(s[i] & 0x80)){
             switch(s[i]){
                 case '\"':PUTS(c, "\\\"", 2);break;
                 case '\\':PUTS(c, "\\\\", 2);break;
                 case '/':PUTC(c, '/');break;
                 case '\b':PUTS(c, "\\b", 2);break;
                 case '\f':PUTS(c, "\\f", 2);break;
                 case '\n':PUTS(c, "\\n", 2);break;
                 case '\r':PUTS(c, "\\r", 2);break;
                 case '\t':PUTS(c, "\\t", 2);break;
                 default:
                 if(s[i] < 0x20){
                   sprintf(lept_context_push(c,6), "\\u00%.2x",(unsigned char)s[i]);
                 }else{
                     PUTC(c,s[i]);
                 }break;
             }

~~~

另外，\uXXXX只能表示4位hexcode，而UTF-8的范围还存在U+10000~U+10FFFF的情况，因此json中采用一组代理对（surrogate pair）\uXXXX\uYYYY表示在U+10000~U+10FFFF范围的码点。如果第一个码点是 U+D800 至 U+DBFF，我们便知道它的代码对的高代理项（high surrogate），之后应该伴随一个 U+DC00 至 U+DFFF 的低代理项（low surrogate）。然后，我们用下列公式把代理对 (H, L) 变换成真实的码点：

~~~
codepoint = 0x10000 + (H − 0xD800) × 0x400 + (L − 0xDC00)
~~~

要把这样的码点还原为原来的字符的话，首先分析该式，发现H表示了高10，L表示了码点的低10位，再加上0x10000,组成21位的码点。因此对该类列码点的还原过程应是先将码点减去0x10000,再取高10位作为H,低10位作为L，而后输出`\uH\uL`。
