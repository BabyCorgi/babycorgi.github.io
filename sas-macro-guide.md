<!-- title: SAS Macro Development Guide -->

<p id="top"></p>

<h2 id="table-of-contents"> Table of Contents </h2>

- [General](#general)
- [Macro Structure](#macro-structure)
  - [1. 程序头 Program Header](#1-程序头-program-header)
  - [2. 宏定义 %macro](#2-宏定义-macro)
    - [macro-name](#macro-name)
    - [macro-parameter](#macro-parameter)
  - [3. 宏开始标记](#3-宏开始标记)
  - [4. 宏参数及宏变量初始化](#4-宏参数及宏变量初始化)
  - [5. 设置默认值](#5-设置默认值)
  - [6. 宏参数核查](#6-宏参数核查)
  - [7. 宏的主体部分](#7-宏的主体部分)
  - [8. 宏结束部分](#8-宏结束部分)


## General

- 符合**GPP**规范
- 只能有ASCII，不包含任何中文字符、特殊字符，保证不同编码环境中(SASApp, SASApp_DBCS, SASApp_UTF8)均能正确运行
- 缩进使用**4个空格**
- 每行不超过120字符，方便code review
    > 潜在风险：注意有些语句、函数对换行敏感，如prx系列函数
- 使用相对路径(灵活使用`init`中配置的全局宏变量)
- 标识符名称(数据集名、数据集变量名、宏变量名、宏参数名等)，名字体现内容含义，并参考以下格式
    - 每个单词首字母大写：`MacroParameter`
    - 单词间用下划线连接：`macro_paramerter` (只使用小写字母)

1. 宏参数及宏变量
    - `local`

2. 中间数据集
    - start with 3 underscores (_) `___dataset`
        > 潜在风险：
    - 注意不要覆盖原有的数据集
    - `___inds_01`, `___inds_02`
    - 注意要方便debug
    - `Debug=1`

3. 数据集变量
    - 注意输入数据集中可能存在的变量名

4. 输出信息
    - 宏输出提示到Log，避免直接写 `ERROR`, `WARNING`，可能会被check log工具误认
    - 加上`(&SYSMACRONAME)`，方便区分此信息是由SAS系统提示还是宏的提示  

    - 宏语句输出Log

        `%put ERR%str(OR): (&SYSMACRONAME) ....;`  
        `%put WARN%str(ING): (&SYSMACRONAME) ....;`  
        `%put NOTES: (&SYSMACRONAME) ....;`  
        `%put (&SYSMACRONAME) ....;`  

    - DATA步输出Log

        `put "ERR" "OR: (&SYSMACRONAME) ....";`  
        `put "WARN" "ING: (&SYSMACRONAME) ....";`

5. 注释
    - Use `%** ;` or `/** */` comment style for comments that do not show in MPRINT output
    - Use `** ;` comment style for comments that show in MPRINT output

[*back to top*](#top)


## Macro Structure

### 1. 程序头 Program Header

```SAS
/*************************************************************************************************
File name:      <macro-name-vx.x>.sas

Study:          

SAS version:    9.4

Purpose:        

Macros called:  

Notes:          

Parameters:     

Sample:         

Date started:   DDMMMYYYY
Date completed: DDMMMYYYY

Mod     Date            Name            Description
---     -----------     ------------    -----------------------------------------------
1.0     DDMMMYYYY       Name.Surname    Create

************************************ Prepared by GCP Clinplus ************************************/
```

与一般项目程序的区别：
- `File name:`  注意每次版本更新要修改版本号  
    
- `Study:`      留空
- `Parameters:` 必填
- `Sample:`     必填

[*back to top*](#table-of-contents)

### 2. 宏定义 %macro

```SAS
%macro macro_name(Param1=    /** Required, xxxx */
                ,Param2 =    /** Optional, xxxx */
                ,Debug  = 
                );
```

#### macro-name
- 文件名与宏名对应，文件名hyphen(-)对应宏名underscore(_)
    > 例如：宏名为%macro_name， 文件名应为 macro-name.sas

#### macro-parameter

- 不在`%macro`句中定义默认值
    > 潜在风险：
- 可以在宏参数后加上简要说明，如`/** Required, xxxx */`
    > 注意：不能使用`%** xxx;`或`* xx;`
- 一般都需要有宏参数`Debug=`，选择是否保留中间数据集，1-是，0-否

[*back to top*](#table-of-contents)

### 3. 宏开始标记

```SAS
%put Start Running Macro (&SYSMACRONAME vx.x);
```

- 标记宏开始，方便debug
- 使用系统宏变量`SYSMACRONAME`标记当前运行的宏名字
- 加上版本号`vx.x`
    > 注意每次版本更新要修改版本号

[*back to top*](#table-of-contents)

### 4. 宏参数及宏变量初始化

```SAS
%local i j total nQueryCheck chinese_mvarlist SQL_mvarlist nSQL_Select_Var;
```

- 声明宏变量为`local`
- `global`
- `call symputx('macro-variable', 'value', "L");`

[*back to top*](#table-of-contents)

### 5. 设置默认值

```SAS
%if %length(&MacroParam1)=0 and %symexist(_program_path) %then %let MacroParam1=&_program_path.;
%if %length(&MacroParam2)=0                              %then %let MacroParam2=xxxxx;
```

宏内使用中文或特殊符号可参考以下代码

```SAS
%let chinese_mvarlist = chn_comma chn_period chn_double_quote_left chn_double_quote_right;
%local &chinese_mvarlist;

data _null_;
    array _chn_des[*] $500 &chinese_mvarlist;

    %** Set macro variable value (DBCS);
    chn_comma = byte(163)||byte(172);
    chn_period = byte(161)||byte(163);
    chn_double_quote_left = byte(161)||byte(176);
    chn_double_quote_right = byte(161)||byte(177);

    %** Define local macro variable;
    do i=1 to dim(_chn_des);
        %if "%upcase(&SYSENCODING)"="UTF-8" %then %do;
            _chn_des[i] = kcvt(_chn_des[i], "EUC-CN", "UTF-8");
        %end;
        call symputx(vname(_chn_des[i]), vvalue(_chn_des[i]), "L");
    end;
run;
```

[*back to top*](#table-of-contents)

### 6. 宏参数核查

需要对输入的宏参数进行必要的核查，避免之后程序报错，同时输出必要且足够的信息至Log，有助于debug  
以下列举一些常见的核查及参考代码

- 宏参数/宏变量是否为空
    ```SAS
    %if %length(&MacroParam1)=0 %then %do;
        %put ERR%str(OR): (&SYSMACRONAME) Macro parameter MacroParam1= should not be null.;
        %goto EXIT;
    %end;
    ```
- SAS数据集是否存在
    ```SAS
    %if %sysfunc(exist(&Dataset))=0 %then %do;
        %put ERR%str(OR): (&SYSMACRONAME) Specified Dataset= does not exist.;
        %goto EXIT;
    %end;
    ```
- SAS数据集观测数`%nobs()`
- SAS数据集中某变量是否存在 `%m_var_exist()`
- SAS数据集中某变量类型`%m_var_type()`

- 文件路径/外部文件是否存在
    ```SAS
    %if %sysfunc(fileexist(&ExternalFile))=0 %then %do;
        %put ERR%str(OR): (&SYSMACRONAME) Specified file/Directory does not exist [&ExternalFile];
        %goto EXIT;
    %end;
    ```

[*back to top*](#table-of-contents)

### 7. 宏的主体部分

```SAS
%** Delete Temporary Datasets;
proc datasets nolist nowarn lib=work memtype=data;
    delete ___:;
quit;
```

根据每个宏的目的不尽相同  
主要考虑以下几点：  
1. 对于输入数据的预处理(包括对输入数据的核查)
2. 核心的代码
3. 对于输出结果格式的调整处理
4. 最终输出

[*back to top*](#table-of-contents)

### 8. 宏结束部分

- 删除中间数据集
- 还原一些设定
- 输出信息至Log

```SAS
%EXIT:
    %if "&Debug"^="1" %then %do;
        proc datasets nolist nowarn lib=work memtype=data;
            delete ___:;
        quit;
    %end;
```

[*back to top*](#table-of-contents)

---

Contributing  
@NG
