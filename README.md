# SQL-DIE
Make sql engineers die

![image](https://user-images.githubusercontent.com/17697154/227728953-72a29d0d-31a3-4ef9-8314-ddc3bca53908.png)


## langchain怎么理解SQL自动化的

```python
"""
Given an input question, first create a syntactically correct {dialect} query to run, then look at the results of the query and return the answer. Unless the user specifies in his question a specific number of examples he wishes to obtain, always limit your query to at most {top_k} results. You can order the results by a relevant column to return the most interesting examples in the database.
Never query for all the columns from a specific table, only ask for a the few relevant columns given the question.
Pay attention to use only the column names that you can see in the schema description. Be careful to not query for columns that do not exist. Also, pay attention to which column is in which table.
Use the following format:
Question: "Question here"
SQLQuery: "SQL Query to run"
SQLResult: "Result of the SQLQuery"
Answer: "Final answer here"
Only use the tables listed below.
{table_info}
Question: {input}
"""
```

```
给定一个输入问题，首先创建一个语法正确的{方言}查询来运行，然后查看查询结果并返回答案。除非用户在问题中指定了他想获得的具体例子数量，否则总是将查询限制在最多{top_k}个结果。您可以按相关列对结果进行排序，以返回数据库中最有趣的示例。
从特定表中永远不要查询所有列，只询问有关问题的几个相关列。
注意只使用模式描述中可以看到的列名。小心不要查询不存在的列。此外，注意哪个列位于哪个表中。
使用以下格式：
问题：“问题在这里”
SQL查询：“要运行的SQL查询”
SQL结果：“SQL查询的结果”
答案：“最终答案在这里”
只使用下面列出的表。
{table_info}
问题：{input}

注：在翻译时，{dialect}和{top_k}需要替换为实际方言和数字。
```

```python
Given the below input question and list of potential tables, output a comma separated list of the table names that may be necessary to answer this question.
Question: {query}
Table Names: {table_names}
Relevant Table Names:
```
