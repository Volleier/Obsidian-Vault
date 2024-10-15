# 公式标记
Markdown两种输入公式的方法：一是行内公式（inline），用一对美元符号“$”包裹。二是整行公式（displayed），用一对紧挨的两个美元符号\$\$包裹

这是一个行内公式: 前文字 $(X^2)^2 = X^{2*2}$ 后文字
写法是：`$(X^2)^2 = X^{2*2}$`

这是一个整行公式：
$$log_2n\frac{1}{n} = -1log_2n$$
写法是：`$$log_2n\frac{1}{n} = -1log_2n$$`

# 特殊字符
| 名称 | 大写 | 写法 | 小写 | 写法 |
| ---- | ---- | ---- | ---- | ---- |
| alpha | A | A | α | \alpha |
| beta | B | B | β | \beat |
| gamma | Γ | Gamma | γ | \gamma |
| delta | Δ | \Delta | δ | \delta |
| epsilon | E | E | ϵ | \epsilon |
| zeta | Z | Z | ζ | \zeta |
| eta | H | H | η | \eta |
| theta | Θ | \Theta | θ | \theta |
| iota | I | I | ι | \iota |
| kappa | K | K | κ | \kappa |
| lambda | Λ | \Lambda | λ | \lambda |
| mu | M | M | μ | \mu |
| nu | N | N | ν | \nu |
| xi | Ξ | \Xi | ξ | \xi |
| omicron | O | O | ο | \omicron |
| pi	|Π | \Pi | π | \pi |  |
| rho | P | P | ρ | \rho |
| sigma | Σ | \Sigma | σ	|\sigma |  |
| tau | T | T | τ | \tau |
| upsilon | Υ | \Upsilon | υ | \upsilon |
| phi | Φ | \Phi | ϕ | \phi |
| chi | X | X | χ | \chi |
| psi | Ψ | \Psi | ψ | \psi |
| omega | Ω | \Omega | ω | \omega |

# 上标/下标
上标和下标分别使用^和_来实现
Example：
\$x_i^2$ -> $x_i^2$​
\$log_2^n$ -> $log_2^x$
默认情况下，上下标符号仅仅对下一个字符作用。一组字符使用{}包裹起来的内容。同时，大括号还能消除二义性，如$x^5^6$ 会显示错误，必须使用大括号来界定^ 的结合性
Example:
\${x^X}^Y$ -> ${x^5}^6$ 
OR
\$x^{X^Y}$ -> $x^{5^6}$ 
另外，如果要在左右两边都有上下标，可以写为 \${^1_2}A{^3_4}$ -> ${^1_2}A{^3_4}$

# 括号
* 小括号与方括号：使用原始的()和[]即可
	Example: \$(2+3)[4+4]$ -> $(2+3)[4+4]$
* 大括号：由于大括号{}被用来分组，因此需要使用"\"转义字符\{和\}表示大括号
	Example: \$\{a\*b\}$ -> $\{a*b\}$
* 尖括号：使用\langle和\rangle分别表示左尖括号和右尖括号
	Example: \$\langle x \rangle$ -> $\langle x \rangle$ 
* 上取整：使用\lceil和\rceil表示
	Example: \$\lceil x \rceil$ $\lceil x \rceil$
* 下取整：使用\lfloor和\rfloor表示
	Example: \$\lfloor x \rfloor$ -> $\lfloor x \rfloor$
需要注意的是，原始括号并不会随着公式的大小自动缩放。
可以使用\left( …\right)来自适应的调整括号
	Example:
	原始括号：
	\$(\frac12)$ -> $(\frac12)$ 
	适应括号：
	 \$\left( \frac12 \right)$ -> $\left( \frac12 \right)$
 
 # 求和/积分
\sum用来表示求和符号，其下标表示求和下限，上标表示上线
	Example: \$\sum_1^n$ -> $\sum_1^n$
\int用来表示积分符号，同样地，其上下标表示积分的上下限
	Example: \$\int_1^\infty$ -> $\int_1^\infty$ 
与此类似的符号还有：
\$\prod$ -> $\prod$
\$\bigcup$ -> $\bigcup$
\$\bigcap$ -> $\bigcap$
\$\iint$ -> $\iint$

# 分式/根式
* 第一种：\$\frac ab$ -> $\frac ab$​
	 如果分子或分母不是单个字符，需要使用{}来分组。
* 第二种：使用\over来分隔一个组的前后两部分，如\${a+1\over b+1}$ -> ${a+1\over b+1}$
	根式使用 \sqrt[a]{b} 来表示。其中，方括号内的值用来表示开几次方，大括号则表示开方
		Example: \$\sqrt[4]{\frac xy}$ ->$\sqrt[4]{\frac xy}$

# 字体

`<font face=“黑体”>我是黑体字</font>`
<font face=“黑体”>我是黑体字</font>
`<font face=“微软雅黑”>我是微软雅黑`
<font face=“微软雅黑”>我是微软雅黑
`<font face=“STCAIYUN”>我是华文彩云</font>`
<font face=“STCAIYUN”>我是华文彩云</font>
`<font color=red>我是红色</font>`
<font color=red>我是红色</font>
`<font color=#008000>我是绿色</font>`
<font color=#008000>我是绿色</font>
`<font color=Blue>我是蓝色</font>`
<font color=Blue>我是蓝色</font>
`<font size=5>我是尺寸</font>`
<font size=5>我是尺寸</font>
`<font face=“黑体” color=green size=5>我是黑体，绿色，尺寸为5</font>`
<font face=“黑体” color=green size=5>我是黑体，绿色，尺寸为5</font>

# 居中
语法：  
`<center>居中<br><br></center > `
效果：
<center> 居中<br><br></center >  