# FlinkSQL - 验证过程


<!--more-->
当Sql语句转换为SqlNode 之后 就会进入下一个阶段：将SqlNode 转换成 Operations
```java
@Override
    public List<Operation> parse(String statement) {
        CalciteParser parser = calciteParserSupplier.get();
        FlinkPlannerImpl planner = validatorSupplier.get();

        Optional<Operation> command = EXTENDED_PARSER.parse(statement);
        if (command.isPresent()) {
            return Collections.singletonList(command.get());
        }

        // parse the sql query
        // use parseSqlList here because we need to support statement end with ';' in sql client.
        SqlNodeList sqlNodeList = parser.parseSqlList(statement);
        List<SqlNode> parsed = sqlNodeList.getList();
        Preconditions.checkArgument(parsed.size() == 1, "only single statement supported");
        return Collections.singletonList(
                SqlNodeToOperationConversion.convert(planner, catalogManager, parsed.get(0))
                        .orElseThrow(() -> new TableException("Unsupported query: " + statement)));
    }
```
其中  SqlNodeToOperationConversion.convert方法，就算是将 SqlNode转换成 Operation  
SqlNod会先进行验证工作，然后再进行转换

```java
    public static Optional<Operation> convert(
            FlinkPlannerImpl flinkPlanner, CatalogManager catalogManager, SqlNode sqlNode) {
        // validate the query
        final SqlNode validated = flinkPlanner.validate(sqlNode);
        return convertValidatedSqlNode(flinkPlanner, catalogManager, validated);
    }
```

## 为什么 SqlNode 需要验证

目的当然是为了确保SQL语句的正确性和合法性，避免在后续的执行过程中出现错误
举例来说，假设有以下的SQL语句：
```sql
SELECT name, age FROM student WHERE age > 18 AND gender = 'male'
```
在进行转换成Operation的过程中，需要先进行validate，检查SQL语句是否符合语法规则和语义规则。如果SQL语句中存在语法错误或者语义错误，那么validate过程会抛出异常，提示用户需要修改SQL语句
例如，如果SQL语句中存在以下错误：
```sql
SELECT name, age FROM student WHERE age > 'two' AND gender = 'male'
```
其中，age的类型是整数，但是在SQL语句中使用了字符串类型的'two'进行比较，这是一个语义错误。在进行validate的过程中，会检测到这个错误并抛出异常，提示用户需要修改SQL语句中的比较条件
因此，先进行validate的过程可以帮助用户在SQL语句执行之前就发现错误，避免在后续的执行过程中出现问题，提高SQL语句的执行效率和准确性

## validate

validate的过程的输入和输出都是SqlNode，除了检查类型之外还会更改SqlNode的节点信息  
通过FlinkPlanner 对SqlNode进行检查  

```java
  def validate(sqlNode: SqlNode): SqlNode = {
    val validator = getOrCreateSqlValidator()
    validate(sqlNode, validator)
  }
```

FlinkCalciteSqlValidator 实现 了Calicte 的SqlValidatorImpl 作为validator，就是用来检查FlinkSql的Sql语法的  

```java
/** Extends Calcite's {@link SqlValidator} by Flink-specific behavior. */
@Internal
public final class FlinkCalciteSqlValidator extends SqlValidatorImpl {

    // Enables CallContext#getOutputDataType() when validating SQL expressions.
    private SqlNode sqlNodeForExpectedOutputType;
    private RelDataType expectedOutputType;

    public FlinkCalciteSqlValidator(
            SqlOperatorTable opTab,
            SqlValidatorCatalogReader catalogReader,
            RelDataTypeFactory typeFactory,
            SqlValidator.Config config) {
        super(opTab, catalogReader, typeFactory, config);
    }

```

### 创建 validator

第一次写入Sql时，会新建一个validator，后面再创建就会使用一个单例来处理

```scala
  def getOrCreateSqlValidator(): FlinkCalciteSqlValidator = {
    if (validator == null) {
      val catalogReader = catalogReaderSupplier.apply(false)
      validator = createSqlValidator(catalogReader)
    }
    validator
  }
```
其中，catalogReaderSupplier是一个函数，目的是读取和解析数据库中的元信息，比如数据库，表，列等信息  
函数的输入是一个bool值，用来确定大小写是否敏感；输出是FlinkCalciteCatalogReader  
FlinkCalciteCatalogReader相比于CalciteCatalogReader，包含了Flink特有的信息，例如：Flink的UDF函数信息和Flink的Table信息

#### FlinkCalciteSqlValidator

通过数据元信息，就可以创建 SqlValidator
```scala
  private def createSqlValidator(catalogReader: CalciteCatalogReader) = {
    val validator = new FlinkCalciteSqlValidator(
      operatorTable,
      catalogReader,
      typeFactory,
      SqlValidator.Config.DEFAULT
        .withIdentifierExpansion(true)
        .withDefaultNullCollation(FlinkPlannerImpl.defaultNullCollation)
        .withTypeCoercionEnabled(false)
    ) // Disable implicit type coercion for now.
    validator
  }
```
创建SqlValidator 需要以下信息

- SqlOperatorTable, 定义了Sql操作符和方法，也可以查找这些操作符和方法

这个SqlOperatorTable是在创建 StreamTableEnvironment的时候就创建了，具体是在
```java
// org.apache.flink.table.planner.delegation.PlannerContext
    public FrameworkConfig createFrameworkConfig() {
        return Frameworks.newConfigBuilder()
                .defaultSchema(rootSchema.plus())
                .parserConfig(getSqlParserConfig())
                .costFactory(new FlinkCostFactory())
                .typeSystem(typeSystem)
                .convertletTable(FlinkConvertletTable.INSTANCE)
                .sqlToRelConverterConfig(getSqlToRelConverterConfig())
                .operatorTable(getSqlOperatorTable(getCalciteConfig())) // 这里创建
                // set the executor to evaluate constant expressions
                .executor(
                        new ExpressionReducer(
                                context.getTableConfig(), context.getClassLoader(), false))
                .context(context)
                .traitDefs(traitDefs)
                .build();
    }
```
一般来说 opTab 里包含的都是 Calcite自带的操作符 比如 not like, is not null 这类

- SqlValidatorCatalogReader, 就是上面所说的FlinkCalciteCatalogReader，保存的SQL的元数据
- RelDataTypeFactory, FlinkSQL的类型，连接 Flink 的 LogicalType 和 Calcite 的 RelDataType。 RelDataType 是 RelNode关系代数中的数据类型，使用Calcite解析SQL的时候内部会转换成这种类型

```java
/**
 * Flink specific type factory that represents the interface between Flink's [[LogicalType]] and
 * Calcite's [[RelDataType]].
 */
class FlinkTypeFactory(
    classLoader: ClassLoader,
    typeSystem: RelDataTypeSystem = FlinkTypeSystem.INSTANCE)
  extends JavaTypeFactoryImpl(typeSystem)
  with ExtendedRelTypeFactory {

  private val seenTypes = mutable.HashMap[LogicalType, RelDataType]()

  /**
   * Create a calcite field type in table schema from [[LogicalType]]. It use PEEK_FIELDS_NO_EXPAND
   * when type is a nested struct type (Flink [[RowType]]).
   *
   * @param t
   *   flink logical type.
   * @return
   *   calcite [[RelDataType]].
   */
  def createFieldTypeFromLogicalType(t: LogicalType): RelDataType = {
    def newRelDataType(): RelDataType = t.getTypeRoot match {
      case LogicalTypeRoot.NULL => createSqlType(NULL)
      case LogicalTypeRoot.BOOLEAN => createSqlType(BOOLEAN)
      case LogicalTypeRoot.TINYINT => createSqlType(TINYINT)
      case LogicalTypeRoot.SMALLINT => createSqlType(SMALLINT)
      case LogicalTypeRoot.INTEGER => createSqlType(INTEGER)
      case LogicalTypeRoot.BIGINT => createSqlType(BIGINT)
      case LogicalTypeRoot.FLOAT => createSqlType(FLOAT)
      case LogicalTypeRoot.DOUBLE => createSqlType(DOUBLE)
      case LogicalTypeRoot.VARCHAR => createSqlType(VARCHAR, t.asInstanceOf[VarCharType].getLength)
      case LogicalTypeRoot.CHAR => createSqlType(CHAR, t.asInstanceOf[CharType].getLength)

      // temporal types
      case LogicalTypeRoot.DATE => createSqlType(DATE)
      case LogicalTypeRoot.TIME_WITHOUT_TIME_ZONE => createSqlType(TIME)

      // interval types
      case LogicalTypeRoot.INTERVAL_YEAR_MONTH =>
        createSqlIntervalType(
          new SqlIntervalQualifier(TimeUnit.YEAR, TimeUnit.MONTH, SqlParserPos.ZERO))
      case LogicalTypeRoot.INTERVAL_DAY_TIME =>
        createSqlIntervalType(
          new SqlIntervalQualifier(TimeUnit.DAY, TimeUnit.SECOND, SqlParserPos.ZERO))

      case LogicalTypeRoot.BINARY => createSqlType(BINARY, t.asInstanceOf[BinaryType].getLength)
      case LogicalTypeRoot.VARBINARY =>
        createSqlType(VARBINARY, t.asInstanceOf[VarBinaryType].getLength)

      case LogicalTypeRoot.DECIMAL =>
        t match {
          case decimalType: DecimalType =>
            createSqlType(DECIMAL, decimalType.getPrecision, decimalType.getScale)
          case legacyType: LegacyTypeInformationType[_]
              if legacyType.getTypeInformation == BasicTypeInfo.BIG_DEC_TYPE_INFO =>
            createSqlType(DECIMAL, 38, 18)
        }

      case LogicalTypeRoot.ROW =>
        val rowType = t.asInstanceOf[RowType]
        buildStructType(
          rowType.getFieldNames,
          rowType.getChildren,
          // fields are not expanded in "SELECT *"
          StructKind.PEEK_FIELDS_NO_EXPAND)

      case LogicalTypeRoot.STRUCTURED_TYPE =>
        t match {
          case structuredType: StructuredType => StructuredRelDataType.create(this, structuredType)
          case legacyTypeInformationType: LegacyTypeInformationType[_] =>
            createFieldTypeFromLogicalType(
              PlannerTypeUtils.removeLegacyTypes(legacyTypeInformationType))
        }

      case LogicalTypeRoot.ARRAY =>
        val arrayType = t.asInstanceOf[ArrayType]
        createArrayType(createFieldTypeFromLogicalType(arrayType.getElementType), -1)

      case LogicalTypeRoot.MAP =>
        val mapType = t.asInstanceOf[MapType]
        createMapType(
          createFieldTypeFromLogicalType(mapType.getKeyType),
          createFieldTypeFromLogicalType(mapType.getValueType))

      case LogicalTypeRoot.MULTISET =>
        val multisetType = t.asInstanceOf[MultisetType]
        createMultisetType(createFieldTypeFromLogicalType(multisetType.getElementType), -1)

      case LogicalTypeRoot.RAW =>
        t match {
          case rawType: RawType[_] =>
            new RawRelDataType(rawType)
          case genericType: TypeInformationRawType[_] =>
            new GenericRelDataType(genericType, true, getTypeSystem)
          case legacyType: LegacyTypeInformationType[_] =>
            createFieldTypeFromLogicalType(PlannerTypeUtils.removeLegacyTypes(legacyType))
        }

      case LogicalTypeRoot.SYMBOL =>
        createSqlType(SqlTypeName.SYMBOL)

      case _ @t =>
        throw new TableException(s"Type is not supported: $t")
    }

    // Kind in TimestampType do not affect the hashcode and equals, So we can't put it to seenTypes
    val relType = t.getTypeRoot match {
      case LogicalTypeRoot.TIMESTAMP_WITHOUT_TIME_ZONE =>
        val timestampType = t.asInstanceOf[TimestampType]
        timestampType.getKind match {
          case TimestampKind.ROWTIME => createRowtimeIndicatorType(t.isNullable, false)
          case TimestampKind.REGULAR => createSqlType(TIMESTAMP, timestampType.getPrecision)
          case TimestampKind.PROCTIME =>
            throw new TableException(
              s"Processing time indicator only supports" +
                s" LocalZonedTimestampType, but actual is TimestampType." +
                s" This is a bug in planner, please file an issue.")
        }
      case LogicalTypeRoot.TIMESTAMP_WITH_LOCAL_TIME_ZONE =>
        val lzTs = t.asInstanceOf[LocalZonedTimestampType]
        lzTs.getKind match {
          case TimestampKind.PROCTIME => createProctimeIndicatorType(t.isNullable)
          case TimestampKind.ROWTIME => createRowtimeIndicatorType(t.isNullable, true)
          case TimestampKind.REGULAR =>
            createSqlType(TIMESTAMP_WITH_LOCAL_TIME_ZONE, lzTs.getPrecision)
        }
      case _ =>
        seenTypes.get(t) match {
          case Some(retType: RelDataType) => retType
          case None =>
            val refType = newRelDataType()
            seenTypes.put(t, refType)
            refType
        }
    }

    createTypeWithNullability(relType, t.isNullable)
  }
  ```
- SqlValidator.Config, SqlValidator的配置信息，这里默认使用的是 ImmutableSqlValidator
```java
   SqlValidator.Config DEFAULT = ImmutableSqlValidator.Config.builder()
        .withTypeCoercionFactory(TypeCoercions::createTypeCoercion)
        .build();
```

### 开始验证

创建validator 后 根据validate 来验证 sqlNode

```
{{<mermaid>}}
graph TB
start(("开始")) --> A("预处理重写SqlNode") --> B("如果有扩展节点，先处理扩展验证")
B --> C{"是否是DDL等"} 
C --Y--> ends("结束")
C --N-->D("根据不同的Sql语句进行验证") -->E("返回SqlNode")-->ends

{{</mermaid>}}
```

#### 重写
为什么要重写？

重写相当于是在SqlNode进行一些预处理主要作用是检查查询语句是否符合Flink Table的语法规范，并对查询语句进行一些必要的转换和优化，以便更好地支持Flink的执行引擎  

Flink中重写是依靠PreValidateReWriter来实现的
```java
      sqlNode.accept(new PreValidateReWriter(validator, typeFactory))
```
如果是简单的SQL其实不需要重写
```scala
  override def visit(call: SqlCall): Unit = {
    call match {
      case e: SqlRichExplain =>
        e.getStatement match {
          case r: RichSqlInsert => rewriteInsert(r)
          case _ => // do nothing
        }
      case r: RichSqlInsert => rewriteInsert(r)
      case _ => // do nothing
    }
  }
```

#### validator验证SqlNode

如果不是特殊的SQL，应该都会使用 validator来验证SqlNode

```scala
        case _ =>
          validator.validate(sqlNode)
```

```java
// org.apache.calcite.sql.validate.SqlValidatorImpl;
    @Override
    public SqlNode validate(SqlNode topNode) {
        SqlValidatorScope scope = new EmptyScope(this);
        scope = new CatalogScope(scope, ImmutableList.of("CATALOG"));
        final SqlNode topNode2 = validateScopedExpression(topNode, scope);
        final RelDataType type = getValidatedNodeType(topNode2);
        Util.discard(type);
        return topNode2;
    }
```

##### SqlValidatorScope

SqlValidatorScope是Calcite中的一个接口，用于表示SQL语句中的作用域。它包含了当前作用域中可见的所有表、列、函数等信息。
具体来说，SqlValidatorScope中包含了以下信息：
- 当前作用域中可见的所有表，包括别名和表的元数据信息。
- 当前作用域中可见的所有列，包括别名和列的元数据信息。
- 当前作用域中可见的所有函数，包括函数的元数据信息和参数信息。
- 当前作用域中可见的所有变量，包括变量的类型和值。

举例说明，假设有以下SQL语句：
```sql
SELECT a.name, b.salary FROM employee a JOIN salary b ON a.id = b.id WHERE b.salary > 5000;
```
在这个SQL语句中，作用域可以分为以下几个部分：
1. SELECT子句中的作用域，包括a.name和b.salary两个列。
2. FROM子句中的作用域，包括employee和salary两个表。
3. JOIN子句中的作用域，包括a和b两个表的别名。
4. WHERE子句中的作用域，包括b.salary列和5000常量。

在每个作用域中，SqlValidatorScope都会包含相应的表、列、函数等信息，以便进行语法和语义的验证。

SqlValidatorScope 本身也是一个树形结构，根节点是一个空scope  

##### validateScopedExpression

验证过程的第一步就是重写SqlNode，将SqlNode中的不确定的地方都进行更新  
然后如果是Select这种简单的SQL，会将查询进行注册


```
{{<mermaid>}}
graph TB

A(("开始")) --> B("SqlNode重写") -->C("注册查询")
X --> Z(("结束"))

{{</mermaid>}}
```



```java
private SqlNode validateScopedExpression(SqlNode topNode, SqlValidatorScope scope) {
        SqlNode outermostNode = performUnconditionalRewrites(topNode, false);
        cursorSet.add(outermostNode);
        top = outermostNode;
        TRACER.trace("After unconditional rewrite: {}", outermostNode);
        if (outermostNode.isA(SqlKind.TOP_LEVEL)) {
            registerQuery(scope, null, outermostNode, outermostNode, null, false);
        }
        outermostNode.validate(this, scope);
        if (!outermostNode.isA(SqlKind.TOP_LEVEL)) {
            // force type derivation so that we can provide it to the
            // caller later without needing the scope
            deriveType(scope, outermostNode);
        }
        TRACER.trace("After validation: {}", outermostNode);
        return outermostNode;
    }
```

**performUnconditionalRewrites**

```
{{<mermaid>}}
graph TB

A(("开始")) --> B("如果SqlNode是SqlCall") -->|是|C("获取SqlNode所有子SqlNode(算子)") --> C1("遍历算子，递归调用得到重写的算子,并更新") -->|递归| B
C1 --> C2("如果SqlNode的操作符是函数") -->|是|C3("查找内部函数并替换") --> C4("如果配置重写，则操作符更新")
B -->|否|D{"是否是SqlNodeList"} -->|是|E("遍历List,递归调用重写，并更新")-->|递归| B
C4 --> D
E -->F{"判断SqlNode的Kind类型"}
F -->|"VALUES，ORDER_BY等其他关键字"|G("根据不同的关键字规则重写") -->H("返回SqlNode") --> Z(("结束"))
F -->|"其他(如Select)"|H

{{</mermaid>}}
```
这里大部分都是递归调用，能够实际重写的大部分都是函数调用
```java
if (call.getOperator() instanceof SqlUnresolvedFunction) {
                assert call instanceof SqlBasicCall;
                final SqlUnresolvedFunction function = (SqlUnresolvedFunction) call.getOperator();
                // This function hasn't been resolved yet.  Perform
                // a half-hearted resolution now in case it's a
                // builtin function requiring special casing.  If it's
                // not, we'll handle it later during overload resolution.
                final List<SqlOperator> overloads = new ArrayList<>();
                opTab.lookupOperatorOverloads(
                        function.getNameAsId(),
                        function.getFunctionType(),
                        SqlSyntax.FUNCTION,
                        overloads,
                        catalogReader.nameMatcher());
                if (overloads.size() == 1) {
                    ((SqlBasicCall) call).setOperator(overloads.get(0));
                }
}
```
SqlNode的lookupOperatorOverloads方法主要是用于查找可用的操作符重载。具体来说，它会根据传入的操作符名称和参数类型，查找符合条件的操作符重载方法。

举个例子，假设有以下的SqlNode：
```sql
SELECT * FROM table WHERE column1 + column2 = 10
```
在这个SqlNode中，有一个加法操作符“+”，它的左右两边分别是column1和column2这两个列。因此，lookupOperatorOverloads方法会首先根据“+”操作符名称查找可用的操作符重载方法。如果找到了多个重载方法，它会根据参数类型进一步筛选出符合条件的重载方法。  

假设我们定义了以下的操作符重载方法：

```java
public static int operator +(int a, int b) {...}
public static double operator +(double a, double b) {...}
public static string operator +(string a, string b) {...}
```
在这种情况下，lookupOperatorOverloads方法会根据column1和column2的数据类型来选择合适的重载方法。
- 如果column1和column2都是int类型，那么会选择第一个重载方法；
- 如果column1和column2都是double类型，那么会选择第二个重载方法；
- 如果column1和column2都是string类型，那么会选择第三个重载方法。  

如果找不到符合条件的操作符重载方法，lookupOperatorOverloads方法会抛出异常。  

**registerQuery**

```
{{<mermaid>}}
graph TB

A(("开始")) --> B("区分不同的nodeKind") -->|SELECT|C("创建并注册Select命名空间") -->D("创建Select作用域")
D --> E("注册where子句") --> F("注册from子句") --> G("创建并注册From Identifier命名空间") --> H("设置TableScope") --> I("Select的SqlNode设置新的From的Node")
I --> J("如果有聚合方法,需要分别注册groupByScope, HavingScope") --> K("如果有orderby还需要注册OrderByScope")
K --> Z(("结束"))

{{</mermaid>}}
```

```java
 private void registerQuery(
            SqlValidatorScope parentScope,
            @Nullable SqlValidatorScope usingScope,
            SqlNode node,
            SqlNode enclosingNode,
            @Nullable String alias,
            boolean forceNullable,
            boolean checkUpdate) {
              ...
            }
```



enclosingNode和Node都是AST节点，但是它们的含义和使用方式有所不同。  
enclosingNode表示当前节点所在的最近的语法结构节点，例如SELECT语句中的FROM子句，WHERE子句等。enclosingNode可以通过调用getEnclosingNode方法获取。  
Node表示当前节点本身，例如SELECT语句中的SELECT子句，FROM子句等。Node可以通过调用getNode方法获取。  
举个例子，假设有以下SQL语句：
```sql
SELECT a, b FROM table1 WHERE a > 10
```
在validate过程中，对于SELECT子句中的a和b，它们的enclosingNode都是SELECT语句本身，而它们的Node分别是ColumnRef节点，表示一个列引用。  
对于WHERE子句中的a > 10，它的enclosingNode是SELECT语句的WHERE子句，而它的Node是一个比较操作符节点，表示一个大于号操作。  
在validate过程中，Calcite会对每个节点进行类型检查、语法检查等操作，以确保SQL语句的正确性。在这个过程中，enclosingNode和Node都会被用到，以确定当前节点的上下文和语义。  

**validate**

1. 检查验证状态，如果没有验证，则开始验证，然后验证类型，Select子句会调用SelectNamespace的验证，最后根据不同的子句分别验证
2. 验证对表的访问(只有from子句才会真正被验证)
3. 验证快照到表
4. 验证Select和其子句的形态 Relation 还是 Stream 

Modality指的是数据流的模式，即数据流的类型和结构。在Relational模式下，数据流是以表格的形式呈现，每个数据流都有一个固定的列集合和数据类型。而在Streaming模式下，数据流是以流的形式呈现，数据是按照时间顺序到达的，每个数据流都是一个无限的数据流。  

举例来说，如果我们有一个查询语句：
```sql
SELECT name, age FROM employees WHERE age > 30
```
- 在Relational模式下，我们需要知道employees表格的列集合和数据类型，以及age列的数据类型是什么，才能验证这个查询语句是否合法。
- 而在Streaming模式下，我们需要知道employees表格的列集合和数据类型，以及数据流是按照时间顺序到达的，才能验证这个查询语句是否合法。  

验证过程中我们需要根据数据流的模式来验证查询语句是否合法。如果数据流的模式与查询语句不匹配，就会抛出异常。  
例如，在Relational模式下，如果查询语句中引用了不存在的列，就会抛出异常；在Streaming模式下，如果查询语句中使用了聚合函数，就会抛出异常，因为数据流是无限的，无法计算聚合函数的结果。  

## 举例
以 
```sql
select * from tableA where amount > 2
```
为例，
```
{{<mermaid>}}
graph BT

C("SelectScope") --> B("CatalogScope") --> A("EmptyScope")
E("tableA 命名空间") --> D("TableScope") --> B


{{</mermaid>}}
```


---

> 作者: toxi  
> URL: https://example.com/flink-sql-validate/  

