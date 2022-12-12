数据库需要管理很多元数据，所谓元数据就是用来描述数据表结构信息的数据。例如在mysql中使用show tables命令，它会把所有表的名称显示出来，这里数据库表的名称就属于元数据。我们要实现的元数据管理包含四部分，分别为表元数据管理，视图元数据管理，索引元数据管理，和统计相关元数据管理。

首先我们在项目中创建文件夹metadata_management, 然后创建文件interface.go用于定义管理器的接口，首先我们定义的是数据表元数据管理器接口，代码如下：

package metadata_manager

import (
    rm "record_manager"
    "tx"
)

type TableManagerInterface interface {
    CreateTable(tblName string, sch *rm.Schema, tx *tx.Transation)
    GetLayout(tblName string, tx *tx.Transation) *rm.Layout
}
表管理器使用上面两个接口来创建数据库表，同时存储用于描述数据表的元数据。每个数据库表都会对应两个表用于存储其元数据，第一个表叫tblcat，它的记录包含两个字段，一个是字符串类型，字段名称为“tblename”,用于存储它所描述的数据库表的名称，一个字段是整形，字段名为slotsize，用于描述目标数据库表一条记录的长度。第二个表叫fldcat，它用来存储所创建表每个字段的元数据，它包含5个字段，第一个字段名为tblname,类型是字符串，用来记录字段所在的表名字，第二个字段名为fldname,类型为字符串，用来记录字段的名称；第三个字段叫type，类型为int，记录字段类型；第四个字段名为length，记录字段数据长度，类型为int；第五个字段名为offset，记录字段在记录中的偏移，类型为int，这些信息在后续的代码实现中会变得清晰。

在table_manager.go中添加表管理器的实现代码：

package metadata_manager

import (
    rm "record_manager"
    "tx"
)

const (
    MAX_NAME = 16
)

type TableManager struct {
    tcatLayout *rm.Layout
    fcatLayout *rm.Layout
}

func NewTableManager(isNew bool, tx *tx.Transation) *TableManager {
    tableMgr := &TableManager{}
    tcatSchema := rm.NewSchema()
    //创建两个表专门用于存储新建数据库表的元数据
    tcatSchema.AddStringField("tblname", MAX_NAME)
    tcatSchema.AddIntField("slotsize")
    tableMgr.tcatLayout = rm.NewLayoutWithSchema(tcatSchema)

    fcatSchema := rm.NewSchema()
    fcatSchema.AddStringField("tblname", MAX_NAME)
    fcatSchema.AddStringField("fldname", MAX_NAME)
    fcatSchema.AddIntField("type")
    fcatSchema.AddIntField("length")
    fcatSchema.AddIntField("offset")
    tableMgr.fcatLayout = rm.NewLayoutWithSchema(fcatSchema)

    if isNew {
        //如果当前数据表是第一次创建，那么为这个表创建两个元数据表
        tableMgr.CreateTable("tblcat", tcatSchema, tx)
        tableMgr.CreateTable("fldcat", fcatSchema, tx)
    }

    return tableMgr
}

func (t *TableManager) CreateTable(tblName string, sch *rm.Schema, tx *tx.Transation) {
    //在创建数据表前先创建tblcat, fldcat两个元数据表
    layout := rm.NewLayoutWithSchema(sch)
    tcat := rm.NewTableScan(tx, "tblcat", t.tcatLayout)
    tcat.Insert()
    tcat.SetString("tblname", tblName)
    tcat.SetInt("slotsize", layout.SlotSize())
    tcat.Close()
    fcat := rm.NewTableScan(tx, "fldcat", t.fcatLayout)
    for _, fldName := range sch.Fields() {
        fcat.Insert()
        fcat.SetString("tblname", tblName)
        fcat.SetString("fldname", fldName)
        fcat.SetInt("type", int(sch.Type(fldName)))
        fcat.SetInt("length", sch.Length(fldName))
        fcat.SetInt("offset", layout.Offset(fldName))
    }
    fcat.Close()
}

func (t *TableManager) GetLayout(tblName string, tx *tx.Transation) *rm.Layout {
    //获取给定表的layout结构
    size := -1
    tcat := rm.NewTableScan(tx, "tblcat", t.tcatLayout)
    for tcat.Next() {
        //找到给定表对应的元数据表
        if tcat.GetString("tblname") == tblName {
            size = tcat.GetInt("slotsize")
            break
        }
    }
    tcat.Close()
    sch := rm.NewSchema()
    offsets := make(map[string]int)
    fcat := rm.NewTableScan(tx, "fldcat", t.fcatLayout)
    for fcat.Next() {
        if fcat.GetString("tblname") == tblName {
            fldName := fcat.GetString("fldname")
            fldType := fcat.GetInt("type")
            fldLen := fcat.GetInt("length")
            offset := fcat.GetInt("offset")
            offsets[fldName] = offset
            sch.AddField(fldName, rm.FIELD_TYPE(fldType), fldLen)
        }
    }
    fcat.Close()
    return rm.NewLayout(sch, offsets, size)
}
在上面代码中，TableManager对象在创建时会先创建两张数据库表分别名为tblcat和tdlcat，同时使用前面实现的接口来设置这两个表的记录结构和字段信息，以后TableManager每创建一个新表时，就会把新表的元数据存储在这两张表中。从代码也可以看出数据库表的元数据其实对应两部分信息，一部分是表所包含的字段信息，一部分是表对应的schema信息，这些信息会作为tblcat和tdlcat这两张表的记录存储起来。

下面我们看看如何小于TableManager提供的接口，在main.go中输入代码如下：

func main() {
    file_manager, _ := fm.NewFileManager("recordtest", 400)
    log_manager, _ := lm.NewLogManager(file_manager, "logfile.log")
    buffer_manager := bmg.NewBufferManager(file_manager, log_manager, 3)

    tx := tx.NewTransation(file_manager, log_manager, buffer_manager)
    sch := record_mgr.NewSchema()
    sch.AddIntField("A")
    sch.AddStringField("B", 9)

    tm := mm.NewTableManager(true, tx)
    tm.CreateTable("MyTable", sch, tx)
    layout := tm.GetLayout("MyTable", tx)
    size := layout.SlotSize()
    sch2 := layout.Schema()
    fmt.Printf("MyTable has slot size: %d\n", size)
    fmt.Println("Its fields are: ")
    for _, fldName := range sch2.Fields() {
        fldType := ""
        if sch2.Type(fldName) == record_mgr.INTEGER {
            fldType = "int"
        } else {
            strlen := sch2.Length(fldName)
            fldType = fmt.Sprintf("varchar( %d )", strlen)
        }
        fmt.Printf("%s : %s", fldName, fldType)
    }

    tx.Commit()
}
在代码中，我们首先创建一个表名为MyTable，它的记录包含两个字段分别是整形字段，名称为A,以及字符串字段，名称为B。在使用CreateTable创建表时，这些信息就已经被写入到tblcat和tdlcat两张表中。

然后代码调用TableManager的GetLayout接口获取表MyTable的结构信息，由于这些信息已经写入两张元数据库表，因此这些信息只要从表里面读取即可。在GetLayout的实现中，它首先根据表的名称”MyTable”在tblcat表中查询这个表一条记录的字节大小，然后从tldcat表中查询它记录所包含的字段信息，这些信息获取后再调用NewLayout接口把MyTable表的Layout结构创建出来。

上面代码运行后输出结果如下：

MyTable has slot size: 33
Its fields are: 
A : int
B : varchar( 9 )
A : int
B : varchar( 9 )
transation 1  committed
从输出可以看到，所谓元数据管理其本质就是创建两个特定的表名为tblcat,tldcat，将新创建的数据表记录的长度以及字段信息分别存储在这两个表中，以后在实现表的管理时，从这两张表中再去查询给定表的layout信息，代码下载链接: https://pan.baidu.com/s/1B6tjAdUsIDXdufdTcRv1pw
提取码: d2iv,
更多内容请在B站搜索Coding迪斯尼.

