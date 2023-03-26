# SQL-DIE
Make sql engineers die

![image](https://user-images.githubusercontent.com/17697154/227728953-72a29d0d-31a3-4ef9-8314-ddc3bca53908.png)


## langchain怎么理解SQL自动化的

不足：这两个问题都要问OPENAI的，只有表名信息不足以体现表的关联，字段等信息


**如何写SQL**
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

**如何找到表**
```python
Given the below input question and list of potential tables, output a comma separated list of the table names that may be necessary to answer this question.
Question: {query}
Table Names: {table_names}
Relevant Table Names:
```

```
给定以下输入问题和潜在表列表，输出可能需要回答此问题的表名称的逗号分隔列表。
问题：{query}
表名：{table_names}
相关表名：
```

**如何执行**
`decider_chian` = 找表
`sql_chain` = 生成 + 执行

decider_chain -> sql_chain.

```python
class SQLDatabaseSequentialChain(Chain, BaseModel):
    """Chain for querying SQL database that is a sequential chain.
    The chain is as follows:
    1. Based on the query, determine which tables to use.
    2. Based on those tables, call the normal SQL database chain.
    This is useful in cases where the number of tables in the database is large.
    """

    return_intermediate_steps: bool = False

    @classmethod
    def from_llm(
        cls,
        llm: BaseLanguageModel,
        database: SQLDatabase,
        query_prompt: BasePromptTemplate = PROMPT,
        decider_prompt: BasePromptTemplate = DECIDER_PROMPT,
        **kwargs: Any,
    ) -> SQLDatabaseSequentialChain:
        """Load the necessary chains."""
        sql_chain = SQLDatabaseChain(
            llm=llm, database=database, prompt=query_prompt, **kwargs
        )
        decider_chain = LLMChain(
            llm=llm, prompt=decider_prompt, output_key="table_names"
        )
        return cls(sql_chain=sql_chain, decider_chain=decider_chain, **kwargs)

    decider_chain: LLMChain
    sql_chain: SQLDatabaseChain
    input_key: str = "query"  #: :meta private:
    output_key: str = "result"  #: :meta private:

    @property
    def input_keys(self) -> List[str]:
        """Return the singular input key.
        :meta private:
        """
        return [self.input_key]

    @property
    def output_keys(self) -> List[str]:
        """Return the singular output key.
        :meta private:
        """
        if not self.return_intermediate_steps:
            return [self.output_key]
        else:
            return [self.output_key, "intermediate_steps"]

    def _call(self, inputs: Dict[str, str]) -> Dict[str, str]:
        _table_names = self.sql_chain.database.get_table_names()
        table_names = ", ".join(_table_names)
        llm_inputs = {
            "query": inputs[self.input_key],
            "table_names": table_names,
        }
        table_names_to_use = self.decider_chain.predict_and_parse(**llm_inputs)
        self.callback_manager.on_text(
            "Table names to use:", end="\n", verbose=self.verbose
        )
        self.callback_manager.on_text(
            str(table_names_to_use), color="yellow", verbose=self.verbose
        )
        new_inputs = {
            self.sql_chain.input_key: inputs[self.input_key],
            "table_names_to_use": table_names_to_use,
        }
        return self.sql_chain(new_inputs, return_only_outputs=True)

    @property
    def _chain_type(self) -> str:
        return "sql_database_sequential_chain"
```























