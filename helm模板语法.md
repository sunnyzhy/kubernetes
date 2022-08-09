# helm 模板语法

## 表达式

- ```{{ 模板表达式 }}```

- ```{{- 模板表达式 -}}```

***注: 横杠 ```-``` 表示去掉表达式输出结果前面和后面的空格，去掉前面空格可以这么写 ```{{- 模板表达式 }}```, 去掉后面空格 ```{{ 模板表达式 -}}```***

## 变量和变量作用域

默认情况最左面的点 ```.```, 代表全局作用域，用于引用全局对象。

示例:

```yaml
## 引用内置对象 Values 中的 key 属性
{{ .Values.key }}
```

helm 全局作用域中有两个重要的内置对象：```Values``` 和 ```Release```

### Values

```Values``` 表示 ```values.yaml``` 定义的参数，通过 ```.Values``` 可以引用 ```values.yaml``` 中的任意参数。

示例:

```yaml
{{ .Values.replicaCount }}

{{ .Values.image.repository }}
```

### Release

```Release``` 表示应用发布时带有的属性。

- Release.Name: release 的名字，一般通过 ```Chart.yaml``` 定义，或者通过 helm 命令在安装应用的时候指定
- Release.Time: release 安装时间
- Release.Namespace: k8s 命名空间
- Release.Revision: release 版本号，是一个递增值，每次更新都会加一
- Release.IsUpgrade: true 代表，当前 release 是一次更新操作
- Release.IsInstall: true 代表，当前 release 是一次安装操作

示例:

```yaml
{{ .Release.Name }}
```


自定义模板变量:

```yaml
{{- $relname := .Release.Name -}}
```

引用自定义变量:

```yaml
## 不需要使用符号 . 来引用
{{ $relname }}
```

##  函数和管道运算符

### 函数运算符

```yaml
{{ functionName arg1 arg2… }}
```

示例:

```yaml
## 调用 quote 函数，将结果用""双引号包括起来
{{ quote .Values.favorite.food }}
```

### 管道（pipelines）运算符 |

通过管道 ```|``` 将多个命令串起来，处理模板输出的内容。

示例:

```yaml
## 将 .Values.favorite.food 传递给 quote 函数处理，然后在输出结果
{{ .Values.favorite.food | quote  }}

## 先将 .Values.favorite.food 的值传递给 uppe r函数将字符转换成大写，然后专递给 quote 加上引号包括起来
{{ .Values.favorite.food | upper | quote }}

## 如果 .Values.favorite.food 为空，则使用 default 定义的默认值
{{ .Values.favorite.food | default "默认值" }}

## 将 .Values.favorite.food 输出 5 次
{{ .Values.favorite.food | repeat 5 }}

## 对输出结果缩进 2 个空格
{{ .Values.favorite.food | nindent 2 }}
```

常用的关系运算符在 helm 模板中都以函数的形式实现:

- ```eq```: ```=```

- ```ne```: ```!=```

- ```lt```: ```<=```

- ```gt```: ```>=```

- ```and```: ```&&```

- ```or```: ```||```

- ```not```: ```!```

示例:

```yaml
## 相当于 if (.Values.fooString && (.Values.fooString == "foo"))
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}
```

## 流程控制语句

### if else

语法:

```yaml
{{ if 条件表达式 }}
## Do something
{{ else if 条件表达式 }}
## Do something else
{{ else }}
## Default case
{{ end }}
```

示例:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
```

### with

```with``` 主要就是用来修改 ```.``` 作用域的；默认 ```.``` 代表全局作用域，```with``` 语句可以修改 ```.``` 的含义。

语法:

```yaml
{{ with 引用的对象 }}
## 这里可以使用 . (点)， 直接引用 with 指定的对象
{{ end }}
```

示例:

```yaml
## .Values.favorite 是一个 object 类型
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}   #相当于.Values.favorite.drink
food: {{ .food | upper | quote }}
{{- end }}
```
 

注: 这里面 ```.``` 相当于已经被重定义了。所以不能在 ```with``` 作用域内使用 ```.``` 引用全局对象。如果非要在 ```with``` 里面引用全局对象，可以先在 ```with``` 外面将全局对象复制给一个变量，然后在 ```with``` 内部使用这个变量引用全局对象。如下：

```yaml
## 先将值保存起来
{{- $release:= .Release.Name -}}

{{- with .Values.favorite }}
## 相当于 .Values.favorite.drink
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}

## 间接引用全局对象的值
release: {{ $release }}
{{- end }}
```

### range

```range``` 主要用于循环遍历数组类型。

- 语法 1:

    ```yaml
    ## 遍历 map 类型，用于遍历键值对象
    ## 变量 key 代表对象的属性名，val 代表属性值
    {{- range key,val := 键值对象 }}
    {{ $key }}: {{ $val | quote }}
    {{- end}}
    ```

- 语法 2:

    ```yaml
    {{- range 数组 }}

    ## . (点)，引用数组元素值
    {{ . | title | quote }}

    {{- end }}
    ```

示例:

```yaml
## 定义 map 类型
favorite:
  drink: coffee
  food: pizza
 
## 遍历 map 类型
{{- range $key, $val := .Values.favorite }}
{{ $key }}: {{ $val | quote }}
{{- end}} 

## 定义数组类型
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

## 遍历数组类型
{{- range .Values.pizzaToppings}}
{{ . | quote }}
{{- end}}
```

## 子模板定义

一般在 ```_``` (下划线)开头的文件中定义子模板。

- 定义模板:
    ```yaml
    {{ define "模板名字" }}
    ## 模板内容
    {{ end }}
    ```

- 引用模板:
    ```yaml
    {{ include "模板名字" 作用域}}
    ```

示例:

```yaml
## 定义模板
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}

## 引用模板
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    ## 引用 mychart.app 模板内容，并对输出结果缩进 4 个空格
    {{ include "mychart.app" . | nindent 4 }}
data:
  myvalue: "Hello World"
```
