# asn.1

asn.1是一种标记语言，描述通信系统之间和应用程序之间进行交换和结构化的信息。是一种与具体网络环境无关、独立于计算机架构和语言的语法格式。

### 基本类型

- BOOL：布尔值
- INTEGER：任意长度的整数
- REAL：实数值的集合
- BIT STRING：比特为单位的字符串
- OCTET STRING：字节为单位的字符串
- ENUMERATED：枚举
- OBJECT IDENTIFIER：唯一标识ISO/ITU-T定义的对象
- NULL：位置符，出现在复杂类型中，本身没有意义，不需编码

### 复杂类型

- SEQUENCE：有序的不同数据类型集合，类似结构体
- SET：无序的不同数据类型集合，相当于结构体
- CHOICE：选择一组数据类型中的一个，相当于c的union
- SEQUENCE OF：有序的同一数据类型集合
- SET OF：无序的同一数据类型集合

### 数据表示

固有类型大写表示，用户定义类型大写开头且至少包含一个非大写字母

- 类型定义     

```
<type name>::=<type>
自定义类型名::=固有类型
```

如     `Numbed3fRepartsSem::=INTEGER；`

- 子类型定义：将变量限制为特定值或特定范围

```
<suubtype name>::=<type>(<constraint>)
子类型名::=固有类型(特定值或特定范围)
```

如     `RRC-TransactionIdentifier::=INTEGER(0.3)`

- 赋值

```
<value name> <type>::=<value>
变量名 类型名::=值
```

如  `Hum NumberOfReportsSent::=30`



- SEQUENCE定义与赋值

`UserAccout::=SEQUENCE{name PrintableString，password PrintbleString}`

`jessicaAccount UserAeeount::={nanle“jessica”，password“admin”}`

- SEQUENCE OF定义与赋值

`Students::=SEQUENCE OF PrintableString；`

`centralClass Students：：={“Lei”,”Hanmeimei”,”Yuyang”}`

