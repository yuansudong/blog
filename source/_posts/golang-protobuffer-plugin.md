---

title: 聊聊golang中,如何开发一个protobuf plugin
cover: /img/golang/protobuffer_plugin_title.png
subtitle: 聊聊golang中,如何开发一个protobuf plugin
categories: "GO语言"
tags: "GO语言"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: development-manual-jwt
---

以下是此次测试的protobuf文件.

```protobuf
syntax = "proto3";
package service.account.v1;
option go_package = "github.com/yuansudong/protoc-gen-template/gen/gopb/service/account/normal/pbv1;pbv1";
import "google/api/annotations.proto";
// C2SInnerLogin 内部用户登录请求
message C2SInnerLogin {
  // UserName  用户名
  string UserName = 1;
  // Passsword 密码
  string Password = 2;
  // AppID 引用ID
  string AppID = 3;

}
// S2CInnerLogin 内部用户登录响应
message S2CInnerLogin {
  // Token 访问令牌
  string Token = 1;
}
// C2SInnerSignin 内部用户注册请求
message C2SInnerSignin {
  // UserName  用户名
  string UserName = 1;
  // Passsword 密码
  string Password = 2;
  // AppID 引用程序ID
  string AppID = 3;
}
// S2CInnerUserSignin 注册时的响应
message S2CInnerSignin {
  string Token = 1;
}
// Account 账户服务V1版本
service Account {
  // InnerLogin 公司渠道登录
  rpc InnerLogin(C2SInnerLogin) returns (S2CInnerLogin) {
    option (google.api.http) = {
      post: "/v1/inner/login"
      body: "*"
    };
  };
  // InnerSignin 公司渠道注册
  rpc InnerSignin(C2SInnerSignin) returns (S2CInnerSignin) {
    option (google.api.http) = {
      post: "/v1/inner/signin"
      body: "*"
    };
  };
}

```



## 1.解析流程



### 01.protoc解析protobuf文件



```bash
${workspaceRoot}/protoc.exe 
        --proto_path=${workspaceRoot}/protos/v3 
        --go_out=plugins=grpc:${env.GOPATH}/src 
        --template_out=lang=go:import_prefix=cccc:${env.GOPATH}/src 
        ${workspaceRoot}/protos/v3/models/address.proto
```



${workspaceRoot} 等价于工程的根目录

${env.GOPATH} 等价于环境变量中的GOPATH



此处需要注意的是，protobuf的插件有命令要求，必须以protoc-gen-[名称]的格式命名,否则protoc找不到.



其中 --{{xxx}}_out 可以随意增加.



比如:



go_out等价的protobuf的插件名称为protoc-gen-go.



template_out等价的protobuf的名称为protoc-gen-template.



haha_out等价的protobuf的名称为protoc-gen-haha



### 02.protoc-gen-template解析数据



protoc解析完毕后,将序列化之后的二进制数据,以标准输入的方式传递给protoc-gen-template.



protoc-gen-template从标准输入读取二进制数据,读取完毕之后,进行反序列化,得到plugin.proto中的一个message,名为CodeGeneratorRequest



其结构如下



```go
type CodeGeneratorRequest struct {
  FileToGenerate []string `protobuf:"bytes,1,rep,name=file_to_generate,json=fileToGenerate" json:"file_to_generate,omitempty"`
  Parameter *string `protobuf:"bytes,2,opt,name=parameter" json:"parameter,omitempty"`
  ProtoFile []*descriptorpb.FileDescriptorProto `protobuf:"bytes,15,rep,name=proto_file,json=protoFile" json:"proto_file,omitempty"`
  CompilerVersion *Version `protobuf:"bytes,3,opt,name=compiler_version,json=compilerVersion" json:"compiler_version,omitempty"`
}
```



FileToGenerate 进行编译的protobuf的名称,有多少protobuf文件,就有多个文件名,此处就是 ${workspaceRoot}/protos/v3/models/address.proto



Parameter 命令行传入的参数,对于此处就是lang=go;import_prefix=cc ,插件传参的格式如下.



```ABAP

--{{插件名称}}_out=lang={{命令行参数}}:{{生成文件路径}}

比如:

${workspaceRoot}/protoc.exe 
        --proto_path=${workspaceRoot}/protos/v3 
        --template_out=lang=go;import_prefix=cccc:${env.GOPATH}/src 
        ${workspaceRoot}/protos/v3/models/address.proto
其中lang=go;import_prefix=cc 就是命令行传入的参数
```



ProtoFile 将protobuf数据,反序列化成了结构体数组,每个元素包含着一个proto文件的所有数据



CompilerVersion 协议版本号



样例代码如下:



```go
// GetRequest 用于从一个标准输入中,获取一个解析请求,io.Reader等价于os.Stdin
func GetRequest(r io.Reader) (*plugin.CodeGeneratorRequest, error) {
  input, err := ioutil.ReadAll(r)
  if err != nil {
    return nil, fmt.Errorf("读取标准输入失败: %v", err)
  }
  req := new(plugin.CodeGeneratorRequest)
  if err = proto.Unmarshal(input, req); err != nil {
    return nil, fmt.Errorf("读取标准输入失败: %v", err)
  }
  log.Println("参数是:",req.Parameter)
  return req, nil
}
```



运行下列命令



```bash

${workspaceRoot}/protoc.exe 
        --proto_path=${workspaceRoot}/protos/v3 
        --go_out=plugins=grpc:${env.GOPATH}/src 
        --template_out=lang=go;import_prefix=cccc:${env.GOPATH}/src 
        ${workspaceRoot}/protos/v3/models/address.proto
```



输出的结果为



```
2021/03/14 12:48:23 参数是: fix=go;import_prefix=cccc
```



### 03.解析File类型



ProtoFile的类型为descriptor.proto中的FileDescriptorProto.其结构如下.



```go

type FileDescriptorProto struct {
  // Name 文件名
  Name    *string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`       // file name, relative to root of source tree
  // Package 包名
  Package *string `protobuf:"bytes,2,opt,name=package" json:"package,omitempty"` // e.g. "foo", "foo.bar", etc.
  // Dependency 该文件依赖哪些proto文件,比如本例中,依赖就有google/api/annotations.proto
  Dependency []string `protobuf:"bytes,3,rep,name=dependency" json:"dependency,omitempty"` 
  PublicDependency []int32 `protobuf:"varint,10,rep,name=public_dependency,json=publicDependency" json:"public_dependency,omitempty"`
  WeakDependency []int32 `protobuf:"varint,11,rep,name=weak_dependency,json=weakDependency" json:"weak_dependency,omitempty"`
  // MessageType 该proto文件下的message类型
  MessageType []*DescriptorProto        `protobuf:"bytes,4,rep,name=message_type,json=messageType" json:"message_type,omitempty"`
  // EnumType 该proto文件下的enum类型
  EnumType    []*EnumDescriptorProto    `protobuf:"bytes,5,rep,name=enum_type,json=enumType" json:"enum_type,omitempty"`
  // Service 该proto文件下的message类型
  Service     []*ServiceDescriptorProto `protobuf:"bytes,6,rep,name=service" json:"service,omitempty"`
  // Extension 扩展
  Extension   []*FieldDescriptorProto   `protobuf:"bytes,7,rep,name=extension" json:"extension,omitempty"`
  // Options选项,比如go_package就是一个选项
  Options     *FileOptions              `protobuf:"bytes,8,opt,name=options" json:"options,omitempty"`
  // SourceCodeInfo 针对于optional的信息
  SourceCodeInfo *SourceCodeInfo `protobuf:"bytes,9,opt,name=source_code_info,json=sourceCodeInfo" json:"source_code_info,omitempty"`
  // syntax 指定是proto2,还是proto3
  Syntax *string `protobuf:"bytes,12,opt,name=syntax" json:"syntax,omitempty"`
}
```



样例demo代码:



```go

func (r *Registry) Load(req *plugin.CodeGeneratorRequest) error {
  for _, file := range req.GetProtoFile() {
    pkg := GoPackage{   
    Path: r._GoPackagePath(file),
    Name: r._DefaultGoPackageName(file),
    }
    if err := r.ReserveGoPackageAlias(pkg.Name, pkg.Path); err != nil {
      for i := 0; ; i++ {
        alias := fmt.Sprintf("%s_%d", pkg.Name, i)
        if err := r.ReserveGoPackageAlias(alias, pkg.Path); err == nil {
          pkg.Alias = alias
          break
        }
      }
    }
    f := &File{
      FileDescriptorProto: file,
      GoPkg:               pkg,
    }
    r._Files[file.GetName()] = f
    r._RegisterMsg(f, nil, file.GetMessageType())
    r._RegisterEnum(f, nil, file.GetEnumType())
    }
}
```



### 04.解析Message类型



MessageType的元素类型为descriptor.proto中的DescriptorProto message,其结构如下



```go

type DescriptorProto struct {
  // Name 消息名,比如C2SInnerLogin
  Name           *string                           `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // Field 该消息下有哪些字段
  Field          []*FieldDescriptorProto           `protobuf:"bytes,2,rep,name=field" json:"field,omitempty"`
  // Extension 该消息下有哪些扩展
  Extension      []*FieldDescriptorProto           `protobuf:"bytes,6,rep,name=extension" json:"extension,omitempty"`
  // NestedType该消息下有哪些嵌套类型
  NestedType     []*DescriptorProto                `protobuf:"bytes,3,rep,name=nested_type,json=nestedType" json:"nested_type,omitempty"`
  // EnumType 该消息下有哪些枚举类型
  EnumType       []*EnumDescriptorProto            `protobuf:"bytes,4,rep,name=enum_type,json=enumType" json:"enum_type,omitempty"`
  // ExtensionRange 该消息的扩展范围
  ExtensionRange []*DescriptorProto_ExtensionRange `protobuf:"bytes,5,rep,name=extension_range,json=extensionRange" json:"extension_range,omitempty"`
  // OneofDecl  该消息下oneof结构
  OneofDecl      []*OneofDescriptorProto           `protobuf:"bytes,8,rep,name=oneof_decl,json=oneofDecl" json:"oneof_decl,omitempty"`
  // Options 该消息下有哪些选项
  Options        *MessageOptions                   `protobuf:"bytes,7,opt,name=options" json:"options,omitempty"`
  // ReservedRange 预留范围
  ReservedRange  []*DescriptorProto_ReservedRange  `protobuf:"bytes,9,rep,name=reserved_range,json=reservedRange" json:"reserved_range,omitempty"`
  // ReservedName 预留名称
  ReservedName []string `protobuf:"bytes,10,rep,name=reserved_name,json=reservedName" json:"reserved_name,omitempty"`
}

```



Field 的元素类型为descriptor.proto中的FieldDescriptorProto message,其结构如下



```go
type FieldDescriptorProto struct {
  // Name 字段名称
  Name   *string                     `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // Number 字段索引,比如 int64 ha=1,其中1就为Number
  Number *int32                      `protobuf:"varint,3,opt,name=number" json:"number,omitempty"`
  // Label 用于指定该字段是required,repeated,optional 类型
  Label  *FieldDescriptorProto_Label `protobuf:"varint,4,opt,name=label,enum=google.protobuf.FieldDescriptorProto_Label" json:"label,omitempty"`
  // Type 指定字段的类型
  Type *FieldDescriptorProto_Type `protobuf:"varint,5,opt,name=type,enum=google.protobuf.FieldDescriptorProto_Type" json:"type,omitempty"`
  // TypeName 对于字段是message和enum类型的,这个代表和此种类型的名称
  TypeName *string `protobuf:"bytes,6,opt,name=type_name,json=typeName" json:"type_name,omitempty"`
  // Extendee  扩展相关
  Extendee *string `protobuf:"bytes,2,opt,name=extendee" json:"extendee,omitempty"`
  // DefaultValue 字段的默认值
  DefaultValue *string `protobuf:"bytes,7,opt,name=default_value,json=defaultValue" json:"default_value,omitempty"`
  // OneofIndex oneof类型的索引
  OneofIndex *int32 `protobuf:"varint,9,opt,name=oneof_index,json=oneofIndex" json:"oneof_index,omitempty"`
  // JsonName 该字段对应的JSON字符串
  JsonName *string       `protobuf:"bytes,10,opt,name=json_name,json=jsonName" json:"json_name,omitempty"`
  // Options 字段呃选项
  Options  *FieldOptions `protobuf:"bytes,8,opt,name=options" json:"options,omitempty"`
  Proto3Optional *bool `protobuf:"varint,17,opt,name=proto3_optional,json=proto3Optional" json:"proto3_optional,omitempty"`
}
```



样例代码:



```go

// _RegisterMsg 注册message类型
func (r *Registry) _RegisterMsg(file *File, outerPath []string, msgs []*descriptor.DescriptorProto) {
  for i, md := range msgs {
    m := &Message{
      File:            file,
      Outers:          outerPath,
      DescriptorProto: md,
      Index:           i,
    }
    for _, fd := range md.GetField() {
      m.Fields = append(m.Fields, &Field{
        Message:              m,
        FieldDescriptorProto: fd,
      })
    }
    file.Messages = append(file.Messages, m)
    r._Msgs[m.FQMN()] = m
    var outers []string
    outers = append(outers, outerPath...)
    outers = append(outers, m.GetName())
    r._RegisterMsg(file, outers, m.GetNestedType())
    r._RegisterEnum(file, outers, m.GetEnumType())
  }
}
```



### 05.解析枚举类型



EnumType 的元素类型为descriptor.proto中的EnumDescriptorProto

message,其结构如下



```
type EnumDescriptorProto struct {  
  // Name 枚举名称
  Name    *string                     `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // Value 枚举值
  Value   []*EnumValueDescriptorProto `protobuf:"bytes,2,rep,name=value" json:"value,omitempty"`
  // Options 枚举选项
  Options *EnumOptions                `protobuf:"bytes,3,opt,name=options" json:"options,omitempty"`
  // ReservedRange 预留范围
  ReservedRange []*EnumDescriptorProto_EnumReservedRange `protobuf:"bytes,4,rep,name=reserved_range,json=reservedRange" json:"reserved_range,omitempty"`
  // ReservedName 预留名称
  ReservedName []string `protobuf:"bytes,5,rep,name=reserved_name,json=reservedName" json:"reserved_name,omitempty"`
}
```



样例代码如下



```go
type MethodDescriptorProto struct {
  // Name rpc的名称
  Name *string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // InputType rpc中的请求类型
  InputType  *string        `protobuf:"bytes,2,opt,name=input_type,json=inputType" json:"input_type,omitempty"`
  // OutputType rpc中的返回类型
  OutputType *string        `protobuf:"bytes,3,opt,name=output_type,json=outputType" json:"output_type,omitempty"`
  // Options rpc中的选项
  Options    *MethodOptions `protobuf:"bytes,4,opt,name=options" json:"options,omitempty"`
  // ClientStreaming 是否是客户端流模式
  ClientStreaming *bool `protobuf:"varint,5,opt,name=client_streaming,json=clientStreaming,def=0" json:"client_streaming,omitempty"`
  // ServerStreaming 是否是服务端流模式
  ServerStreaming *bool `protobuf:"varint,6,opt,name=server_streaming,json=serverStreaming,def=0" json:"server_streaming,omitempty"`
}
```



### 06.解析service类型



service 的元素类型为descriptor.proto中的ServiceDescriptorProto

 message,其结构如下



```go
type ServiceDescriptorProto struct {
  // Name service的名称
  Name    *string                  `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // Method service包含的rpc方法
  Method  []*MethodDescriptorProto `protobuf:"bytes,2,rep,name=method" json:"method,omitempty"`
  // Options service中包含的选项
  Options *ServiceOptions          `protobuf:"bytes,3,opt,name=options" json:"options,omitempty"`
}
```



service类型中rpc方法的类型为descriptor.proto中的MethodDescriptorProto message,其结构如下



```go
type MethodDescriptorProto struct {
  // Name rpc的名称
  Name *string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
  // InputType rpc中的请求类型
  InputType  *string        `protobuf:"bytes,2,opt,name=input_type,json=inputType" json:"input_type,omitempty"`
  // OutputType rpc中的返回类型
  OutputType *string        `protobuf:"bytes,3,opt,name=output_type,json=outputType" json:"output_type,omitempty"`
  // Options rpc中的选项
  Options    *MethodOptions `protobuf:"bytes,4,opt,name=options" json:"options,omitempty"`
  // ClientStreaming 是否是客户端流模式
  ClientStreaming *bool `protobuf:"varint,5,opt,name=client_streaming,json=clientStreaming,def=0" json:"client_streaming,omitempty"`
  // ServerStreaming 是否是服务端流模式
  ServerStreaming *bool `protobuf:"varint,6,opt,name=server_streaming,json=serverStreaming,def=0" json:"server_streaming,omitempty"`
}
```



样例代码:



```go
func (r *Registry) _LoadServices(file *File) error {
  var svcs []*Service
  for _, sd := range file.GetService() {
    svc := &Service{
      File:                   file,
      ServiceDescriptorProto: sd,
    }
    for _, md := range sd.GetMethod() {
      meth, err := r._NewMethod(svc, md)
      if err != nil {
        return err
      }
      svc.Methods = append(svc.Methods, meth)
    }
    if len(svc.Methods) == 0 {
      continue
    }
    svcs = append(svcs, svc)
  }
  file.Services = svcs
  return nil
}
```



### 07.将要生成的代码数据包装成CodeGeneratorResponse_File



CodeGeneratorResponse_File 位于plugin.proto文件中，其结构如下。



```go
// Represents a single generated file.
type CodeGeneratorResponse_File struct {
    // Name 文件名
	Name *string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
    // InsertionPoint 目前还不知道用途
	InsertionPoint *string `protobuf:"bytes,2,opt,name=insertion_point,json=insertionPoint" json:"insertion_point,omitempty"`
	// Content 文件的内容
	Content *string `protobuf:"bytes,15,opt,name=content" json:"content,omitempty"`
}
```



样例代码如下



```go
// GoTemplate 用于生成相关的模板
func (g *gen) GoTemplate(file *gengo.File) *plugin.CodeGeneratorResponse_File {
	var err error
	as := &Args{}
	as.Imports = make(map[string]bool)
	as.PackageName = file.GoPkg.Name
	as.DateTime = time.Now().Local().String()
	buf := bytes.NewBuffer(make([]byte, 0, 40960))
	rspFile := new(plugin.CodeGeneratorResponse_File)
	for _, service := range file.Services {
		g.GoService(file, as, service)
	}
	if len(as.Services) != 0 {
		as.IsHave = true
	}

	tp := template.New("template.service")
	if tp, err = tp.Parse(codeFileTemplate); err != nil {
		log.Fatalln(err.Error())
	}
	if err = tp.Execute(buf, as); err != nil {
		log.Fatalln(err.Error())
	}
	name, err := g.GetAllFilePath(file)
	if err != nil {
		log.Println(PluginName, err.Error())
		os.Exit(-1)
	}
	ext := filepath.Ext(name)
	base := strings.TrimSuffix(name, ext)
	output := fmt.Sprintf("%s.pb.template.go", base)
	rspFile.Name = proto.String(output)
	rspFile.Content = proto.String(buf.String())
	return rspFile
}
```



### 08.将自定义代码写回给protoc,让protoc生成文件



protobuf写回，是将数据包装成plugin.proto中的plugin.CodeGeneratorResponse结构，是向os.Stdout写入，写入之后会传给protoc,protoc会根据文件数据,将其输出到文件中。

plugin.CodeGeneratorResponse结构如下。

```go
// 用于写一段代码到protoc
type CodeGeneratorResponse struct {
	// Error 用于向protoc返回数据
	Error *string `protobuf:"bytes,1,opt,name=error" json:"error,omitempty"`
	// 目前还不知用途
	SupportedFeatures *uint64                       `protobuf:"varint,2,opt,name=supported_features,json=supportedFeatures" json:"supported_features,omitempty"`
	// 需要生成的文件,即返回的文件可以有多个
    File              []*CodeGeneratorResponse_File `protobuf:"bytes,15,rep,name=file" json:"file,omitempty"`
}

```



将这个结构，通过protobuf序列化之后，将得到的数据写入os.Stdout, 样例代码如下



```go
// WriteResponse 用于向标准输出中写数据
func WriteResponse(rsp *plugin.CodeGeneratorResponse) {
	buf, err := proto.Marshal(rsp)
	if err != nil {
		log.Fatalln(err.Error())
	}
	if _, err := os.Stdout.Write(buf); err != nil {
		log.Fatalln(err.Error())
	}
}
```

至此,全篇解析完毕.

demo地址:github.com/yuansudong/protoc-gen-template