# org.apache.hadoop.cli 模块详解

## 一、概述

`org.apache.hadoop.cli` 是 Hadoop 的命令行接口（CLI）测试框架，用于自动化测试 Hadoop 的命令行工具。该模块提供了一套完整的测试基础设施，包括测试配置解析、命令执行、输出比较等功能。

**核心目标**：
- 自动化测试 Hadoop 命令行工具
- 支持多种输出比较策略
- 提供灵活的测试配置机制
- 支持测试结果汇总和报告

**位置**: `hadoop-common-project/hadoop-common/src/test/java/org/apache/hadoop/cli/`

---

## 二、核心架构

### 2.1 类继承关系

```
CLITestHelper (测试助手)
├── TestConfigFileParser (配置文件解析器)
└── 测试方法
    ├── testAll() - 运行所有测试
    ├── setUp() - 初始化
    └── tearDown() - 清理

CLICommand (命令接口)
├── CLITestCmd (测试命令实现)
└── CLICommandTypes (命令类型标记)
    └── CLICommandFS (文件系统命令类型)

CommandExecutor (命令执行器)
└── FSCmdExecutor (文件系统命令执行器)

ComparatorBase (比较器基类)
├── ExactComparator (精确比较器)
├── RegexpComparator (正则表达式比较器)
├── RegexpAcrossOutputComparator (跨输出正则比较器)
├── SubstringComparator (子串比较器)
└── TokenComparator (令牌比较器)

数据模型
├── CLITestData (测试数据)
├── ComparatorData (比较器数据)
└── CommandExecutor.Result (执行结果)
```

### 2.2 主要组件

```
org.apache.hadoop.cli
├── CLITestHelper.java          - 测试助手类
├── TestCLI.java                - CLI 测试类
├── testConf.xsl                - 测试配置 XSL 样式表
└── util/
    ├── CLICommand.java         - 命令接口
    ├── CLICommandTypes.java   - 命令类型接口
    ├── CLICommandFS.java      - 文件系统命令类型
    ├── CLITestCmd.java        - 测试命令实现
    ├── CommandExecutor.java   - 命令执行器接口
    ├── FSCmdExecutor.java     - 文件系统命令执行器
    ├── CLITestData.java       - 测试数据
    ├── ComparatorData.java    - 比较器数据
    ├── ComparatorBase.java    - 比较器基类
    ├── ExactComparator.java   - 精确比较器
    ├── RegexpComparator.java  - 正则表达式比较器
    ├── RegexpAcrossOutputComparator.java - 跨输出正则比较器
    ├── SubstringComparator.java - 子串比较器
    └── TokenComparator.java   - 令牌比较器
```

---

## 三、主要组件详解

### 3.1 CLITestHelper - 测试助手

**位置**: `CLITestHelper.java:45`

**作用**: CLI 测试的核心助手类，负责测试配置解析、命令执行、结果比较和报告生成

**核心字段**:
```java
// 测试模式
public static final String TESTMODE_TEST = "test";           // 运行测试并比较
public static final String TESTMODE_NOCOMPARE = "nocompare"; // 运行测试不比较
public static final String TEST_CACHE_DATA_DIR =             // 测试缓存目录
    System.getProperty("test.cache.data", "build/test/cache");

// 测试状态
protected String testMode = TESTMODE_TEST;                  // 当前测试模式
protected ArrayList<CLITestData> testsFromConfigFile = null; // 从配置文件读取的测试
protected ArrayList<ComparatorData> testComparators = null;  // 测试比较器
protected ComparatorData comparatorData = null;              // 当前比较器数据
protected Configuration conf = null;                         // 配置
protected String clitestDataDir = null;                     // CLI 测试数据目录
protected String username = null;                           // 用户名
```

**核心方法**:

#### 1. readTestConfigFile - 读取测试配置文件
```java
protected void readTestConfigFile() {
    String testConfigFile = getTestFile();
    if (testsFromConfigFile == null) {
        boolean success = false;
        testConfigFile = TEST_CACHE_DATA_DIR + File.separator + testConfigFile;
        try {
            SAXParser p = XMLUtils.newSecureSAXParserFactory().newSAXParser();
            p.parse(testConfigFile, getConfigParser());
            success = true;
        } catch (Exception e) {
            LOG.info("Exception while reading test config file {}:",
                testConfigFile, e);
            success = false;
        }
        assertTrue(success, "Error reading test config file");
    }
}
```

#### 2. expandCommand - 扩展命令
```java
protected String expandCommand(final String cmd) {
    String expCmd = cmd;
    expCmd = expCmd.replaceAll("CLITEST_DATA", clitestDataDir);
    expCmd = expCmd.replaceAll("USERNAME", username);
    return expCmd;
}
```

#### 3. compareTestOutput - 比较测试输出
```java
private boolean compareTestOutput(ComparatorData compdata, Result cmdResult) {
    String comparatorType = compdata.getComparatorType();
    Class<?> comparatorClass = null;
    
    boolean compareOutput = false;
    
    if (testMode.equals(TESTMODE_TEST)) {
        try {
            // 初始化比较器类并运行其比较方法
            comparatorClass = Class.forName("org.apache.hadoop.cli.util." + 
                comparatorType);
            ComparatorBase comp = (ComparatorBase) comparatorClass.newInstance();
            compareOutput = comp.compare(cmdResult.getCommandOutput(), 
                expandCommand(compdata.getExpectedOutput()));
        } catch (Exception e) {
            LOG.info("Error in instantiating the comparator" + e);
        }
    }
    
    return compareOutput;
}
```

#### 4. compareTextExitCode - 比较退出码
```java
private boolean compareTextExitCode(ComparatorData compdata,
    Result cmdResult) {
    return compdata.getExitCode() == cmdResult.getExitCode();
}
```

#### 5. testAll - 运行所有测试
```java
public void testAll() {
    assertTrue(testsFromConfigFile.size() > 0,
        "Number of tests has to be greater then zero");
    LOG.info("TestAll");
    
    // 运行测试配置文件中定义的测试
    for (int index = 0; index < testsFromConfigFile.size(); index++) {
        CLITestData testdata = testsFromConfigFile.get(index);
    
        // 执行测试命令
        ArrayList<CLICommand> testCommands = testdata.getTestCommands();
        Result cmdResult = null;
        for (CLICommand cmd : testCommands) {
            try {
                cmdResult = execute(cmd);
            } catch (Exception e) {
                fail(StringUtils.stringifyException(e));
            }
        }
        
        boolean overallTCResult = true;
        // 运行比较器
        ArrayList<ComparatorData> compdata = testdata.getComparatorData();
        for (ComparatorData cd : compdata) {
            final String comptype = cd.getComparatorType();
            
            boolean compareOutput = false;
            boolean compareExitCode = false;
            
            if (! comptype.equalsIgnoreCase("none")) {
                compareOutput = compareTestOutput(cd, cmdResult);
                if (cd.getExitCode() == -1) {
                    // 如果未指定退出码，则无需检查
                    compareExitCode = true;
                } else {
                    compareExitCode = compareTextExitCode(cd, cmdResult);
                }
                overallTCResult &= (compareOutput & compareExitCode);
            }
            
            cd.setExitCode(cmdResult.getExitCode());
            cd.setActualOutput(cmdResult.getCommandOutput());
            cd.setTestResult(compareOutput);
        }
        testdata.setTestResult(overallTCResult);
        
        // 执行清理命令
        ArrayList<CLICommand> cleanupCommands = testdata.getCleanupCommands();
        for (CLICommand cmd : cleanupCommands) {
            try { 
                execute(cmd);
            } catch (Exception e) {
                fail(StringUtils.stringifyException(e));
            }
        }
    }
}
```

#### 6. displayResults - 显示结果
```java
private void displayResults() {
    LOG.info("Detailed results:");
    LOG.info("----------------------------------\n");
    
    // 显示失败的测试详情
    for (int i = 0; i < testsFromConfigFile.size(); i++) {
        CLITestData td = testsFromConfigFile.get(i);
        boolean testResult = td.getTestResult();
        
        if (!testResult) {
            LOG.info("-------------------------------------------");
            LOG.info("                    Test ID: [" + (i + 1) + "]");
            LOG.info("           Test Description: [" + td.getTestDesc() + "]");
            LOG.info("");

            ArrayList<CLICommand> testCommands = td.getTestCommands();
            for (CLICommand cmd : testCommands) {
                LOG.info("              Test Commands: [" + 
                    expandCommand(cmd.getCmd()) + "]");
            }

            LOG.info("");
            ArrayList<CLICommand> cleanupCommands = td.getCleanupCommands();
            for (CLICommand cmd : cleanupCommands) {
                LOG.info("           Cleanup Commands: [" +
                    expandCommand(cmd.getCmd()) + "]");
            }

            LOG.info("");
            ArrayList<ComparatorData> compdata = td.getComparatorData();
            for (ComparatorData cd : compdata) {
                boolean resultBoolean = cd.getTestResult();
                LOG.info("                 Comparator: [" + 
                    cd.getComparatorType() + "]");
                LOG.info("         Comparision result:   [" + 
                    (resultBoolean ? "pass" : "fail") + "]");
                LOG.info("            Expected output:   [" + 
                    expandCommand(cd.getExpectedOutput()) + "]");
                LOG.info("              Actual output:   [" + 
                    cd.getActualOutput() + "]");
            }
            LOG.info("");
        }
    }
    
    // 显示汇总结果
    LOG.info("Summary results:");
    LOG.info("----------------------------------\n");
    
    boolean overallResults = true;
    int totalPass = 0;
    int totalFail = 0;
    int totalComparators = 0;
    for (int i = 0; i < testsFromConfigFile.size(); i++) {
        CLITestData td = testsFromConfigFile.get(i);
        totalComparators += testsFromConfigFile.get(i).getComparatorData().size();
        boolean resultBoolean = td.getTestResult();
        if (resultBoolean) {
            totalPass++;
        } else {
            totalFail++;
        }
        overallResults &= resultBoolean;
    }
    
    LOG.info("               Testing mode: " + testMode);
    LOG.info("");
    LOG.info("             Overall result: " + 
        (overallResults ? "+++ PASS +++" : "--- FAIL ---"));
    LOG.info("               # Tests pass: " + totalPass +
        " (" + (100 * totalPass / (totalPass + totalFail)) + "%)");
    LOG.info("               # Tests fail: " + totalFail + 
        " (" + (100 * totalFail / (totalPass + totalFail)) + "%)");
    LOG.info("         # Validations done: " + totalComparators + 
        " (each test may do multiple validations)");
    
    assertTrue(overallResults, "One of the tests failed. " +
        "See the Detailed results to identify " +
        "the command that failed");
}
```

---

### 3.2 TestConfigFileParser - 配置文件解析器

**位置**: `CLITestHelper.java:379`

**作用**: 解析测试配置 XML 文件，提取测试用例信息

**核心字段**:
```java
String charString = null;                          // 字符串缓冲
CLITestData td = null;                             // 当前测试数据
ArrayList<CLICommand> testCommands = null;         // 测试命令列表
ArrayList<CLICommand> cleanupCommands = null;      // 清理命令列表
boolean runOnWindows = true;                       // 是否在 Windows 上运行
```

**核心方法**:

#### 1. startDocument - 文档开始
```java
@Override
public void startDocument() throws SAXException {
    testsFromConfigFile = new ArrayList<CLITestData>();
}
```

#### 2. startElement - 元素开始
```java
@Override
public void startElement(String uri, 
        String localName, 
        String qName, 
        Attributes attributes) throws SAXException {
    if (qName.equals("test")) {
        td = new CLITestData();
    } else if (qName.equals("test-commands")) {
        testCommands = new ArrayList<CLICommand>();
    } else if (qName.equals("cleanup-commands")) {
        cleanupCommands = new ArrayList<CLICommand>();
    } else if (qName.equals("comparators")) {
        testComparators = new ArrayList<ComparatorData>();
    } else if (qName.equals("comparator")) {
        comparatorData = new ComparatorData();
        comparatorData.setExitCode(-1);
    }
    charString = "";
}
```

#### 3. endElement - 元素结束
```java
@Override
public void endElement(String uri, String localName,String qName)
    throws SAXException {
    if (qName.equals("description")) {
        td.setTestDesc(charString);
    } else if (qName.equals("windows")) {
        runOnWindows = Boolean.parseBoolean(charString);
    } else if (qName.equals("test-commands")) {
        td.setTestCommands(testCommands);
        testCommands = null;
    } else if (qName.equals("cleanup-commands")) {
        td.setCleanupCommands(cleanupCommands);
        cleanupCommands = null;
    } else if (qName.equals("command")) {
        if (testCommands != null) {
            testCommands.add(new CLITestCmd(charString, new CLICommandFS()));
        } else if (cleanupCommands != null) {
            cleanupCommands.add(new CLITestCmd(charString, new CLICommandFS()));
        }
    } else if (qName.equals("comparators")) {
        td.setComparatorData(testComparators);
    } else if (qName.equals("comparator")) {
        testComparators.add(comparatorData);
    } else if (qName.equals("type")) {
        comparatorData.setComparatorType(charString);
    } else if (qName.equals("expected-output")) {
        comparatorData.setExpectedOutput(charString);
    } else if (qName.equals("expected-exit-code")) {
        comparatorData.setExitCode(Integer.valueOf(charString));
    } else if (qName.equals("test")) {
        if (!Shell.WINDOWS || runOnWindows) {
            testsFromConfigFile.add(td);
        }
        td = null;
        runOnWindows = true;
    } else if (qName.equals("mode")) {
        testMode = charString;
        if (!testMode.equals(TESTMODE_NOCOMPARE) &&
            !testMode.equals(TESTMODE_TEST)) {
            testMode = TESTMODE_TEST;
        }
    }
}
```

#### 4. characters - 字符内容
```java
@Override
public void characters(char[] ch, 
        int start, 
        int length) throws SAXException {
    String s = new String(ch, start, length);
    charString += s;
}
```

---

### 3.3 CLICommand - 命令接口

**位置**: `CLICommand.java:25`

**作用**: 定义测试命令的通用接口

**核心方法**:
```java
public interface CLICommand {
    // 获取命令执行器
    CommandExecutor getExecutor(String tag, Configuration conf)
        throws IllegalArgumentException;
    
    // 获取命令类型
    CLICommandTypes getType();
    
    // 获取命令字符串
    String getCmd();
    
    // 转换为字符串
    @Override
    String toString();
}
```

---

### 3.4 CLITestCmd - 测试命令实现

**位置**: `CLITestCmd.java:26`

**作用**: 实现测试命令，包含命令字符串和类型

**核心字段**:
```java
private final CLICommandTypes type;  // 命令类型
private final String cmd;            // 命令字符串
```

**核心方法**:
```java
@Override
public CommandExecutor getExecutor(String tag, Configuration conf)
    throws IllegalArgumentException {
    if (getType() instanceof CLICommandFS)
        return new FSCmdExecutor(tag, new FsShell(conf));
    throw new
        IllegalArgumentException("Unknown type of test command: " + getType());
}

@Override
public CLICommandTypes getType() {
    return type;
}

@Override
public String getCmd() {
    return cmd;
}

@Override
public String toString() {
    return cmd;
}
```

---

### 3.5 CommandExecutor - 命令执行器

**位置**: `CommandExecutor.java`

**作用**: 定义命令执行器的接口

**核心方法**:
```java
public interface CommandExecutor {
    // 执行命令
    Result executeCommand(String cmd) throws Exception;
    
    // 执行结果
    class Result {
        private String commandOutput;  // 命令输出
        private int exitCode;          // 退出码
        
        public String getCommandOutput() {
            return commandOutput;
        }
        
        public int getExitCode() {
            return exitCode;
        }
    }
}
```

---

### 3.6 FSCmdExecutor - 文件系统命令执行器

**位置**: `FSCmdExecutor.java`

**作用**: 执行文件系统命令

**核心字段**:
```java
private String tag;          // 标签
private FsShell shell;      // FsShell 实例
```

**核心方法**:
```java
@Override
public Result executeCommand(String cmd) throws Exception {
    Result result = new Result();
    
    // 执行命令
    int exitCode = shell.run(cmd.split(" "));
    
    // 获取输出
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    PrintStream ps = new PrintStream(out);
    System.setOut(ps);
    
    result.setExitCode(exitCode);
    result.setCommandOutput(out.toString());
    
    return result;
}
```

---

### 3.7 CLITestData - 测试数据

**位置**: `CLITestData.java:27`

**作用**: 存储单个测试用例的数据

**核心字段**:
```java
private String testDesc = null;                          // 测试描述
private ArrayList<CLICommand> testCommands = null;        // 测试命令列表
private ArrayList<CLICommand> cleanupCommands = null;     // 清理命令列表
private ArrayList<ComparatorData> comparatorData = null;  // 比较器数据列表
private boolean testResult = false;                      // 测试结果
```

**核心方法**:
```java
// Getter 和 Setter 方法
public String getTestDesc() { return testDesc; }
public void setTestDesc(String testDesc) { this.testDesc = testDesc; }

public ArrayList<CLICommand> getTestCommands() { return testCommands; }
public void setTestCommands(ArrayList<CLICommand> testCommands) { 
    this.testCommands = testCommands; 
}

public ArrayList<CLICommand> getCleanupCommands() { return cleanupCommands; }
public void setCleanupCommands(ArrayList<CLICommand> cleanupCommands) { 
    this.cleanupCommands = cleanupCommands; 
}

public ArrayList<ComparatorData> getComparatorData() { return comparatorData; }
public void setComparatorData(ArrayList<ComparatorData> comparatorData) { 
    this.comparatorData = comparatorData; 
}

public boolean getTestResult() { return testResult; }
public void setTestResult(boolean testResult) { this.testResult = testResult; }
```

---

### 3.8 ComparatorData - 比较器数据

**位置**: `ComparatorData.java:25`

**作用**: 存储比较器的数据

**核心字段**:
```java
private String expectedOutput = null;  // 期望输出
private String actualOutput = null;    // 实际输出
private boolean testResult = false;    // 测试结果
private int exitCode = 0;              // 退出码
private String comparatorType = null;  // 比较器类型
```

**核心方法**:
```java
// Getter 和 Setter 方法
public String getExpectedOutput() { return expectedOutput; }
public void setExpectedOutput(String expectedOutput) { 
    this.expectedOutput = expectedOutput; 
}

public String getActualOutput() { return actualOutput; }
public void setActualOutput(String actualOutput) { 
    this.actualOutput = actualOutput; 
}

public boolean getTestResult() { return testResult; }
public void setTestResult(boolean testResult) { this.testResult = testResult; }

public int getExitCode() { return exitCode; }
public void setExitCode(int exitCode) { this.exitCode = exitCode; }

public String getComparatorType() { return comparatorType; }
public void setComparatorType(String comparatorType) { 
    this.comparatorType = comparatorType; 
}
```

---

### 3.9 ComparatorBase - 比较器基类

**位置**: `ComparatorBase.java:26`

**作用**: 定义比较器的抽象基类

**核心方法**:
```java
public abstract class ComparatorBase {
    public ComparatorBase() {
        
    }
    
    /**
     * 比较方法
     * @param actual 实际输出，可以为 null
     * @param expected 期望输出，可以为 null
     * @return 如果期望输出与实际输出匹配则返回 true，否则返回 false
     *         如果 actual 或 expected 为 null，则返回 false
     */
    public abstract boolean compare(String actual, String expected);
}
```

---

### 3.10 ExactComparator - 精确比较器

**位置**: `ExactComparator.java:28`

**作用**: 精确比较实际输出和期望输出

**核心方法**:
```java
@Override
public boolean compare(String actual, String expected) {
    return actual.equals(expected);
}
```

**使用场景**:
- 需要完全匹配的输出
- 固定格式的输出
- 精确的错误消息

---

### 3.11 RegexpComparator - 正则表达式比较器

**位置**: `RegexpComparator.java:33`

**作用**: 使用正则表达式比较输出，逐行匹配

**核心方法**:
```java
@Override
public boolean compare(String actual, String expected) {
    boolean success = false;
    Pattern p = Pattern.compile(expected);
    
    StringTokenizer tokenizer = new StringTokenizer(actual, "\n\r");
    while (tokenizer.hasMoreTokens() && !success) {
        String actualToken = tokenizer.nextToken();
        Matcher m = p.matcher(actualToken);
        success = m.matches();
    }
    
    return success;
}
```

**使用场景**:
- 输出格式可变
- 需要匹配特定模式
- 时间戳、路径等动态内容

---

### 3.12 RegexpAcrossOutputComparator - 跨输出正则比较器

**位置**: `RegexpAcrossOutputComparator.java:33`

**作用**: 使用正则表达式比较输出，跨行匹配

**核心方法**:
```java
@Override
public boolean compare(String actual, String expected) {
    if (Shell.WINDOWS) {
        actual = actual.replaceAll("\\r", "");
        expected = expected.replaceAll("\\r", "");
    }
    return Pattern.compile(expected).matcher(actual).find();
}
```

**使用场景**:
- 需要匹配跨行模式
- 多行输出匹配
- 复杂的输出格式

---

### 3.13 SubstringComparator - 子串比较器

**位置**: `SubstringComparator.java`

**作用**: 检查期望输出是否包含在实际输出中

**核心方法**:
```java
@Override
public boolean compare(String actual, String expected) {
    return actual.contains(expected);
}
```

**使用场景**:
- 只需要检查部分输出
- 输出内容较多
- 关键信息验证

---

### 3.14 TokenComparator - 令牌比较器

**位置**: `TokenComparator.java:30`

**作用**: 比较期望输出中的每个令牌是否都在实际输出中

**核心方法**:
```java
@Override
public boolean compare(String actual, String expected) {
    boolean compareOutput = true;
    
    StringTokenizer tokenizer = new StringTokenizer(expected, ",\n\r");
    
    while (tokenizer.hasMoreTokens()) {
        String token = tokenizer.nextToken();
        if (actual.indexOf(token) != -1) {
            compareOutput &= true;
        } else {
            compareOutput &= false;
        }
    }
    
    return compareOutput;
}
```

**使用场景**:
- 需要检查多个独立的信息
- 输出顺序不重要
- 列表内容验证

---

### 3.15 CLICommandTypes - 命令类型接口

**位置**: `CLICommandTypes.java`

**作用**: 定义命令类型的标记接口

**实现类**:
- `CLICommandFS` - 文件系统命令类型

---

### 3.16 CLICommandFS - 文件系统命令类型

**位置**: `CLICommandFS.java:20`

**作用**: 标记文件系统命令类型

**实现**:
```java
public class CLICommandFS implements CLICommandTypes {
}
```

---

## 四、工作流程

### 4.1 完整的测试流程

```
1. 初始化
   ↓
2. 读取测试配置文件 (testConf.xml)
   - 使用 SAX 解析器解析 XML
   - 提取测试用例、命令、比较器等信息
   ↓
3. 运行测试
   ↓
4. 对于每个测试用例
   ↓
5. 执行测试命令
   - 创建命令执行器
   - 执行命令
   - 获取输出和退出码
   ↓
6. 运行比较器
   - 根据比较器类型创建比较器实例
   - 比较实际输出和期望输出
   - 比较退出码
   ↓
7. 记录测试结果
   ↓
8. 执行清理命令
   ↓
9. 重复步骤 4-8，直到所有测试完成
   ↓
10. 显示测试结果
    - 详细结果（失败的测试）
    - 汇总结果（通过/失败统计）
    - 失败测试列表
    - 通过测试列表
   ↓
11. 断言整体结果
```

### 4.2 配置文件格式

**测试配置文件结构**:
```xml
<configuration>
  <tests>
    <test>
      <description>测试描述</description>
      <windows>true/false</windows>
      <test-commands>
        <command>测试命令1</command>
        <command>测试命令2</command>
      </test-commands>
      <comparators>
        <comparator>
          <type>比较器类型</type>
          <expected-output>期望输出</expected-output>
          <expected-exit-code>退出码</expected-exit-code>
        </comparator>
      </comparators>
      <cleanup-commands>
        <command>清理命令1</command>
        <command>清理命令2</command>
      </cleanup-commands>
    </test>
  </tests>
  <mode>test/nocompare</mode>
</configuration>
```

**示例**:
```xml
<configuration>
  <tests>
    <test>
      <description>ls:列出目录内容</description>
      <windows>true</windows>
      <test-commands>
        <command>-ls CLITEST_DATA</command>
      </test-commands>
      <comparators>
        <comparator>
          <type>RegexpComparator</type>
          <expected-output>Found 2 items</expected-output>
          <expected-exit-code>0</expected-exit-code>
        </comparator>
      </comparators>
      <cleanup-commands>
        <command>-rm -r CLITEST_DATA/test</command>
      </cleanup-commands>
    </test>
  </tests>
  <mode>test</mode>
</configuration>
```

---

## 五、关键特性

### 5.1 灵活的测试配置

**优势**:
- 使用 XML 配置文件定义测试用例
- 支持多个测试命令和清理命令
- 支持多个比较器
- 支持平台特定的测试（Windows）

**实现**:
- SAX 解析器解析 XML 配置
- 支持变量替换（CLITEST_DATA、USERNAME）
- 支持测试模式切换（test/nocompare）

### 5.2 多种比较策略

**优势**:
- 支持多种输出比较方式
- 适应不同的输出格式
- 灵活的验证机制

**比较器类型**:
- `ExactComparator` - 精确匹配
- `RegexpComparator` - 正则表达式匹配（逐行）
- `RegexpAcrossOutputComparator` - 正则表达式匹配（跨行）
- `SubstringComparator` - 子串匹配
- `TokenComparator` - 令牌匹配

### 5.3 命令执行器

**优势**:
- 支持多种命令类型
- 可扩展的执行器框架
- 统一的执行结果接口

**实现**:
- `CommandExecutor` 接口定义执行器
- `FSCmdExecutor` 实现文件系统命令执行
- 支持自定义执行器

### 5.4 测试结果报告

**优势**:
- 详细的失败信息
- 汇总的统计信息
- 清晰的测试结果展示

**实现**:
- 显示失败的测试详情
- 显示通过/失败统计
- 显示失败测试列表
- 显示通过测试列表

### 5.5 平台支持

**优势**:
- 支持 Windows 和 Unix/Linux
- 平台特定的测试
- 自动处理平台差异

**实现**:
- `windows` 标记控制测试是否在 Windows 上运行
- `RegexpAcrossOutputComparator` 处理 Windows 换行符

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `test.cache.data` | build/test/cache | 测试缓存数据目录 |
| `testMode` | test | 测试模式（test/nocompare） |
| `comparatorType` | - | 比较器类型 |
| `expectedOutput` | - | 期望输出 |
| `expectedExitCode` | -1 | 期望退出码 |

---

## 七、使用示例

### 7.1 创建测试类

```java
public class TestCLI extends CLITestHelper {
    
    @Before
    public void setUp() throws Exception {
        super.setUp();
        this.username = System.getProperty("user.name");
    }
    
    @After
    public void tearDown() throws Exception {
        super.tearDown();
    }
    
    @Override
    protected String getTestFile() {
        return "testConf.xml";
    }
    
    @Override
    protected CommandExecutor.Result execute(CLICommand cmd) throws Exception {
        CommandExecutor executor = cmd.getExecutor("test", conf);
        return executor.executeCommand(expandCommand(cmd.getCmd()));
    }
    
    @Test
    public void testAll() {
        super.testAll();
    }
}
```

### 7.2 创建测试配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <tests>
    <test>
      <description>ls:列出目录内容</description>
      <windows>true</windows>
      <test-commands>
        <command>-ls CLITEST_DATA</command>
      </test-commands>
      <comparators>
        <comparator>
          <type>RegexpComparator</type>
          <expected-output>Found 2 items</expected-output>
          <expected-exit-code>0</expected-exit-code>
        </comparator>
      </comparators>
      <cleanup-commands>
        <command>-rm -r CLITEST_DATA/test</command>
      </cleanup-commands>
    </test>
    
    <test>
      <description>mkdir:创建目录</description>
      <windows>true</windows>
      <test-commands>
        <command>-mkdir CLITEST_DATA/test</command>
      </test-commands>
      <comparators>
        <comparator>
          <type>ExactComparator</type>
          <expected-output></expected-output>
          <expected-exit-code>0</expected-exit-code>
        </comparator>
      </comparators>
      <cleanup-commands>
        <command>-rm -r CLITEST_DATA/test</command>
      </cleanup-commands>
    </test>
  </tests>
  <mode>test</mode>
</configuration>
```

### 7.3 运行测试

```bash
# 运行所有测试
mvn test -Dtest=TestCLI

# 运行单个测试
mvn test -Dtest=TestCLI#testAll

# 使用 nocompare 模式生成期望输出
mvn test -Dtest=TestCLI -Dtest.mode=nocompare
```

---

## 八、总结

`org.apache.hadoop.cli` 模块是 Hadoop 的命令行测试框架，具有以下特点：

1. **灵活性**: 使用 XML 配置文件定义测试用例，支持多种比较策略
2. **可扩展性**: 支持自定义命令执行器和比较器
3. **易用性**: 提供简洁的 API 和详细的测试报告
4. **平台支持**: 支持 Windows 和 Unix/Linux 平台
5. **自动化**: 自动化测试命令行工具，减少手工测试

**关键设计思想**:
- 配置驱动的测试框架
- 多种比较策略适应不同输出格式
- 命令执行器抽象支持多种命令类型
- 详细的测试结果报告便于问题定位
- 平台特定的测试支持

通过深入理解 `org.apache.hadoop.cli` 模块，可以更好地编写和维护 Hadoop 命令行工具的测试用例，提高测试效率和质量。
