# Data-Driven-Manufacturing-test-framework
XML based test framework for hardware.

To test the hardware component, the most intuitional way is to write one test program for each of them. This is easy
to start but not easy to maintain. The code size will bloast dramatically with more component involved.

This is something like procedural programming. In fact, we can create one OOP(object oriented programming) to do the same
thing.

The new data-driven framework unify the logging and test options. The developers no longer need to know the details of
how to send out test command and get the response. They only focus on the logical business specific to each test item.

Below is the comparison table.

|           | old procedural  |  New XML based data-driven framework |  commment |
| --------- | ------- | ---------------- | ------------ |
| Languages |  C, expect, Python | Python | C is used to test RAM and FE switch |
| Coupling  | Very high | Very low | |
| Option to dump the failure reason of last test | No | Yes | extremely useful for factory worker  |
| limits setting for Sensors | No | Yes | proper limits range enhance the boards quality |
| Requirement of dev | Good experience of Python | Little. The idea situation is one only needs to know how to fill up the XML config data without touching core Python framework | even suitable for hardware (not software) worker|
| Requirement of maintenance | Good experience of Python | Little | suitable for intern with a bit hardware knowledge |
| Dev using GUI | No | Partial (XML data could be updated via GUI ) | |
| multiple card inter communication | No | Yes | |
| Dev time | 2 ~ 3 weeks | 2 ~ 3 days | |

## Design Model
Architecture Diagram
![Architecture](https://github.com/scircle/Data-Driven-Manufacturing-test-framework/blob/master/doc/architecture%20%20uml.jpg)

The new framework has the following:
* One Base Class       - htBaseCommand
* One core test engine - htFactory
* Some sub-classes     - htDefault, htDebug, htTelnet, htSensors, htDsl, htBcm and htPots. All inherit from htBaseCommand.
* One log auxiliary class - htLog used to save the summary, debug info and details of test items.
* One verbose log util - htTee used to dump verbose information on console and one detail log.
* One util containing the definition of all warning and error string - htError
* One bug utility program - hstUtil
* Some configuration files (two global INI files and others are boards specific XML files)

## Sequence Diagram
![Call Flow](https://github.com/scircle/Data-Driven-Manufacturing-test-framework/blob/master/doc/Sequence%20Diagram.jpg)

## Design Description
### Root Class
The htTestBase is the root class. It contains some member functions: **preprocess()**, **process()** and **postprocess()**. 
These three big functions are borrowed from the idea of Windows MFC (VC 6.0) and Java.
The **preprocess()** parses the command line options.
The **process()** is actually no-op. It could be re-written to throw one exception if it's no implemented in sub-class.
The **postprocess()** just returns the final checking result (PASSED/FAILED).

### Factory
Its main purpose is to find one appropriate sub-class and dynamically instantiate one to handle the request.
As shown in the sequence diagram above, it goes through following steps:
1) Collects all the options.
2) Check whether there is one XML config file dedicated to this type of board.
3) Use built-in Python library -xml.etree to parse this XML config file.
4) Find one appropriate sub-class/util to instantiate one and calls the thee big three member functions to send out
   command and get the response.
5) Log the summary of this test and the debug message if the checking result is FAILED.

  Please aware that one single linux command is *insufficient* to test some complex tests. In accommodate to these 
  situations, one tricky technology in Python is used. It's **getattr()**. This built-int getattr() Python function is
  used to fecth an attribute from an object, using a **string** object instead of an **identifier**. In other words, it 
  allow us to call methods based on contents of a string instead of typing the method name. This is something simliar to 
  the function pointer in C. Therefore, utility funciton could be used to accomplish complex tests.
  
### Sensors sub-class
  As we know, sensors includes Temperature, Voltage and Current. The output of "sensors -A" is so complex that we
  create the sensor class to parse the output of this linux command.
  
  Besides, user-defined limits could be used to check whether the output of Temp, Vol and Cur and in good range.
  For example, the Vol for one core component is in the range of 3.2 and 3.4.
  
### Log class
  The miLog class (not sub-class) supports the mini log functionality. 
  There are three log files, xxx**Summary.log**, xxx**Detail.log** and xxx**Debug.log**. All are under /tmp directory.
  
  The xxxSummary.log saves the informations of the count of PASS and FAIL along with the start and end time of one test.
  
  The xxxDebug.log saves the error message if the test failed. This helps the developer and tester quickly know why it
  failed last time without running the test again.
  
  The xxxDetail.log is used to save all the run-time information of all tests. After running all tests, we can just
  dump this file once rather than running each test again with verbose mode.
  
### Tee class
  The Tee class (not sub-class) works in a manner very simliar to the linux "tee" command. It not only outputs to 
  STDOUT but also to one detail log (/tmp/xxxDetail.log). By default, it only outputs to the detail log. When one
  specific option - "-v" used, it also outputs to **console** so that users get the **live** output.
  
### Configuration file format
  There are two types of configuration files used. 
  The first is INI and second is XML.
  
 #### INI configuration
  The first INI file is mainly for global configuration. There are two INI files.
  The first one (htGlobalCfg.ini) indicates the boards types and its corresponding XML config file name. e.g.
  EQPT_BOARD_EPON = 3700
  ...
  3700 = htCmdMapping3700.xml
  
  The second type of INI file (htGlobalCmdMap.ini) setups the mapping between test item name (used in command line)
  and internal name in XML files. e.g.
  ```Python
  CPU=CPU
  I2C=I2C
  MGMGPHY=MGMGPHY
  RTC=RTC
  ...
  ```
  
  To test the CPU, issue  the command "htFactory.py -t cpu" (the test name here is case-insensitive). The factory
  program will first search the CPU item in this INI file. If one is found, it continues to find the corresponding
  section in XML file. Take the EPON card for example, its XML file contains the CPU section as shown below:
  ```xml
  <CPU>
    <CMD name="cat /proc/cpuinfo">
     fsl, T1020
  </CPU>
   ```

 For the details of XML configuration, see next section.
 
#### XML configuration
 The second type is XML file.
 Why XML is selected? There are five types of configuration files in Python.
 1. Python code  
    Simplest way to write configuration files. The Python code is treated as data and we just import it.
   
 2. INI  
   It's fine for most basic config. But it doesn't allow deep nesting like XML.
   
 3. JSON  
   It's used mostly in Web.
   
 4. YAML (YAML Ain't Markup Language)  
   It's not built-in Python.
   
 5. XML  
   Easy to read, write and parsing.
   There are several popular, lightweight, featureful and free XML parsing libraries avaiable.
   As some tags or notations needs to be indicates the type of command and the string matching strategy (exact
   maching, contain or regular), JSON and XML become the last two candidates.
   
   Which one is better?
   
   The fundamental difference between them is that XML is a markup language whereas JSON is a way of representing
   objects. One starting point of the new framework design is treating everything as abstract resource. The program
   just blindly executes the instruction configured no matter what the resource is (phycial port of simple linux command).
   
   One XML document contains XML Elements. An XML element is everyting from (including) the element's start tag to
   (including) the element's end tag. An element contains:
     . Attributes
     . Text
     . Other elements
     . Or a mix of the above
     
   Take the CPU section for example again:
   ```xml
   <CPU>
     <CMD name="cat /proc/cpuinfo">
     fsl,T1020
   </CPU>
   ```
          
   Here, the CPU is the element. The <CMD> is the start tag. The arrribute is the "name". The text is "fsl,T1020".
   And the </CMD> is the end tag.
     
   The text is used as expected result. In this case, it's used to compare the response of "cat /proc/cpuinfo" against
   the text. If they matches, it reports PASS. Otherwise, it outputs FAILED.
     
   Besides the name attribute, there are some other attributes and they would be called "keyword" in this framework.
     
   If there is one special character in the command, put the backslash in front of every non-alphanumeric character.
     
   ##### XInclude
   There are several variants of some cards. One example is VDSL. This series contains Data-Only and Overlay cards.
   Accordingly, there are two XML config files. Most part of the content of these two XML are exactly the same (Two
   copies of one test item in two XML files). This highly redundant config makes the XML files bloat.
     
   One good way to resolve this is to make the config data behavior like CPlusPlus code which supports inheritance and
   containing. The spirit of modular programming dictates that you break your tasks up into small manageable chunks.
   This dictum is also applicable when producing XML documents.
     
   There are many ways solving the problem of constructing a single XML document from multiple documents. XML 
   Inclusions (XInclude) is destined to be the universal general-purpose machanism for facilitating modularity. **[1] [2]**
     
   The XInclude mechanism can be used to iincorporate content from other XML files.
     
   XInclude syntax is extremely simple, just two elements in the http://www.w3.org/2001/XInclude namespace, namely
   ***include*** and ***fallback***. The commonly used namespace prefix is "xi".
     
  The xi:include element serves as inclusion instruction. It defines which document to include and how. Its attributes
  used are:
       href - An URI reference of the document to include.
       
   XML inclusion is a recursive process. So each xi:include element in an included document is processed too.
     
   There is one limitation on the depth of recursion in this framework. Up to **two** depth is set as this is enough.
   Example:
       xxx_Overlay includes one xxx_DataOnly.xml which in turn includes one xxx_Base.xml.
       
   ##### XPath
   The XML file is parsed using the parse() in ElementTree of Python.
     
   To find one interesting (matching) element, the findall() is used to find all matching subelements, by tag name or
   path. And it returns a list containing all matching elements in document order.
     
   XPath is used to search one matching test item. If it doens't find one in current XML document, it continues to
   find using XPath mechanism to search all sub-elements on all levels beneath the current element (searching entire
   subtree).
     
   If there are more than one same tag after expansion, the list returned by the findall() contains multiple results.
   e.g.
     There is one RTC tag in xxx_Base.xml and also in xxx_DataOnly.xml. see the sketch below:
     
     root            (xxx_Overlay.xml)
     ...
        root         (xxx_DataOnly.xml)
        ...
          root       (xxx_Base.xml)
          ...
             RTC  tag
          ...
          /root
        ...
        RTC tag
        ...
        /root
     ...
     /root
    
     
   The purpose of two existence of same RTC in two xml files is that the RTC in xxx_DataOnly.xml is supposed to 
   **override** the RTC in xxx_Base.xml (i.e. the sub xml overrides the same tag in root xml).
     
   Executing the RTC twice is not expected. So we only get the **last** element in the list and re-assign to the list
   itself. 
   Note: Getting the last element is based two factors:
     1) All matching elements returned by findall() are in document order.
     2) The base xml (e.g. xxx_Base.xml) is put before another sub-chidren (e.g. xxx_DataOnly.xml), if exists.
     
   #### Keywords in XML
   There are some keywords- "**type**", "**next**", "**returncode**", "**policy**", "**policy_data**", "**wait**"
   and "**cmdargs**".
   Their values are all enclosed by double quote.
   
   **type**  
    Valid values are "BCM", "POTS", "SENSORS" and "UTIL".  
    
   It determines which kind os sub-classes should parse the command. It maps the "BCM", "POTS", "SENSORS and "UTIL"
   to htTestBcm, htTestPots, htTestSensors and htTestCommon sub-classes respectively.  
    
   If "type" is not specified, it maps to htTestCommon by default.
    
   **next**  
     Valid values are "yes" and "no".  
     
   It indicates whether the current command is the last command (i.e. treasts a bunch of commands as one set).
     
   **returncode**  
     Valid value is "ignore".  
     
   In rare case, testers don't care about whether the response matches the expected result. Another example is 
   it doesn't matter if one command return FAIL. One example is the ping command in the traffic test from the CPU
   to BCM Kanata.  
   ```xml
   <CMD name="ping -c 3 172.15.2.1" returncode="ignore">
   </CMD>
   ```
     
   Here, the ping command is just used to send packets out. It returns failure since the Kanata chipset drops the
   packages. We don't care about the response of this ping command. What we do care in this case is the count of
   packets in next BCM debug command.
     
   **Policy**  
     Valid values are "NO", "regex".  
     
   The "NO" is used only for DDR memory test as it checks there is **no** error in all kinds of memory test.
   The "regex" is used to test regular matching. The expected result should be regular expression. It's widely used
   in traffic test to check the count of packets.  
     
   If it's not specified, the program uses the **contain** matching as the default string matching.
     
   **policy_data**  
     Values are flexible.  
     
   It's not used aone. It must be used together with the "NO" policy.
          
   **wait**  
     Its values is the number of seconds.  
     
   If's mainly used in BCM debug commands. If no wait specified in one command, the response of next command would
   not match the expected result as it needs a bit time to execute current command.
     
   **cmdargs**  
     No fixed values.  
     
   Primarily used for user-define limits setting.
     
   ## Steps to support new type boards  
   1) Add one line in CMDMAPPINGFILE section of htGlobalCfg.ini with format like:  
      3610=htCmdMappingX3.xml
      
   2) Update the command mapping in htGlobalCmdMap.ini if new test item needed. e.g.  
      to test usb, add one line:
      USB=USB
      
   3) Constructs the XML file.   
      If the new test item belongs to the existing sub-classes. It's simple to add te "type" keyword indicating
      which sub-class should be called. Otherwise, we might either add one new sub-class (very rarely as current
      sub-classes is sufficient) or add one utility function in the htUtil.py and specify the "type" as "UTIL" 
      for this command.
   
     
   Reference:  
     [1]: https://msdn.microsoft.com/en-us/library/aa302291.aspx  
     [2]: https://en.wikipedia.org/wiki/XInclude
     
