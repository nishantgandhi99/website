<h1>Most Important Expect of Writing Production Grade Software</h1>
<date> mar 27 <date>
**Code Readability: **

Can you write the code which can even understand by developer who does not code in the same language.
Few Important Links:
https://medium.freecodecamp.org/how-to-columnize-your-code-to-improve-readability-f1364e2e77ba

**Use of Logger: **

Use logger to log you activity. Understand different log levels. There is a sweet spot between logging too much and logging too less which you need to understand based on your application. All production grade application have log monitoring systems like ELK Stack. Learn to integrate them using which ever logging library you choose to use.

**Use of Configuration File: **

Never hard code anything which is going to change when you deploy them in different systems for different clients. Examples are Database IP address, Directory Paths, API tokens. 

**Use of Try/Exception: **

Put code into try-exception block whenever things can go wrong.  Most of the cases when you are trying to parse some kind of input or trying to interact with external system like database or web apis.

**Unit Test: **

Unit test is one of the best way to validate your business logic code. The only catch is writing test cases that cover all corner cases for your business logic. Never make assumptions while writing business logic about inputs. Writing extensive unit test is only way to make sure that your code does what it is expected given any conditions. 

**Integration Test: **

In the real word, where everything is SaaS, very rare application operate on stand alone mode and does not communicate with other systems like databases or queue system. The whole application is made of many independent component which communicate via driver or networking protocols. The integration between them must be tested to make sure that communication is not broken. 

**Easy Setup Instruction: **

Production Grade application must provide clear instruction on how to setup the application and how to run it.

**ReadMe.md: **

The Brief introduction about application must be provided by read-me file. It should provide the link for setup instruction. It must include information about licencing. 

**Module Design and Coding Best Practices: **

Every language has coding style specification for standardizing the way we expression the instruction for consistent experience for coders in team. Understand and use them. Like the programming language, there is design standards followed by the team.

**Few Important Links:**
+ Java: Code Conventions:
  + http://www.oracle.com/technetwork/java/codeconvtoc-136057.html
+ Python PEP:
 + https://www.python.org/dev/peps/pep-0008/ 
 + https://www.python.org/dev/peps/pep-0423/ 
+ Scala:
  + http://docs.scala-lang.org/style/index.html

**Other Materials:**
+ https://medium.freecodecamp.org/how-to-read-your-way-to-becoming-a-better-developer-b6432fa5bc0c
