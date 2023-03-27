# SQL-DIE
Make sql engineers die

![image](https://user-images.githubusercontent.com/17697154/227728953-72a29d0d-31a3-4ef9-8314-ddc3bca53908.png)    

**只要你把数据库表设计的特别傻逼, 那谁谁都替代不了你, 你是永恒的**      
还有什么空间：向量搜索， 表推荐， 个性化吧也就。    


## langchain怎么理解SQL自动化的



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


## `Chain = Chain(llm, prompts)`

一系列的prompt和一个llm组成了一个Chain。 

![72DDBC16-C3BE-4FCD-9A8A-41775CBE551E](https://user-images.githubusercontent.com/17697154/227764738-9be1633c-096b-45a2-b1bb-ccc9f4c6030f.png)


## Database Info

Langchain在获取表元信息的时候，还附带了两行数据。     
这个真的牛逼啊， 利用好元数据，能干翻一切了。  
```
CREATE TABLE "Track" (
	"TrackId" INTEGER NOT NULL, 
	"Name" NVARCHAR(200) NOT NULL, 
	"AlbumId" INTEGER, 
	"MediaTypeId" INTEGER NOT NULL, 
	"GenreId" INTEGER, 
	"Composer" NVARCHAR(220), 
	"Milliseconds" INTEGER NOT NULL, 
	"Bytes" INTEGER, 
	"UnitPrice" NUMERIC(10, 2) NOT NULL, 
	PRIMARY KEY ("TrackId"), 
	FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"), 
	FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"), 
	FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)
/*
2 rows from Track table:
TrackId	Name	AlbumId	MediaTypeId	GenreId	Composer	Milliseconds	Bytes	UnitPrice
1	For Those About To Rock (We Salute You)	1	1	1	Angus Young, Malcolm Young, Brian Johnson	343719	11170334	0.99
2	Balls to the Wall	2	2	1	None	342562	5510424	0.99
*/
```



langchain的Database模块，动态构建了数据库建表语句查询，列信息查询，以及示例行查询。   
基本可以说是SQL自动化的典范了。  
```python
tables = []
        for table in meta_tables:
            if self._custom_table_info and table.name in self._custom_table_info:
                tables.append(self._custom_table_info[table.name])
                continue

            # add create table command
            create_table = str(CreateTable(table).compile(self._engine))

            if self._sample_rows_in_table_info:
                # build the select command
                command = select(table).limit(self._sample_rows_in_table_info)

                # save the columns in string format
                columns_str = "\t".join([col.name for col in table.columns])

                try:
                    # get the sample rows
                    with self._engine.connect() as connection:
                        sample_rows = connection.execute(command)
                        # shorten values in the sample rows
                        sample_rows = list(
                            map(lambda ls: [str(i)[:100] for i in ls], sample_rows)
                        )

                    # save the sample rows in string format
                    sample_rows_str = "\n".join(["\t".join(row) for row in sample_rows])

                # in some dialects when there are no rows in the table a
                # 'ProgrammingError' is returned
                except ProgrammingError:
                    sample_rows_str = ""

                table_info = (
                    f"{create_table.rstrip()}\n"
                    f"/*\n"
                    f"{self._sample_rows_in_table_info} rows from {table.name} table:\n"
                    f"{columns_str}\n"
                    f"{sample_rows_str}\n"
                    f"*/"
                )

                # build final info for table
                tables.append(table_info)

            else:
                tables.append(create_table)

        final_str = "\n\n".join(tables)
        return final_str
```

## Flan T5 Vs Codex

FlanT5真是看不懂SQL的    

**Codex**
```
/*
q: what is the purpose of the following table, assuming this is a
mobile game company database.

a: this table is used to store the user information. the data
is used to analyze the user behavior.

q: what is the granularity of the table (what uniquely defins a row)
a: user_id
*/
CREATE TABLE aoz.dim_user_info (
    user_id bigint ENCODE raw distkey,
    user_name character varying(100) ENCODE lzo,
    city_level integer ENCODE az64,
```

**FlanT5**
```
create table user_info(user_id.....)..........................
```


## GPT4
```
The "app_diamond_report" table seems to be a record of user activity within an application that involves virtual diamonds. The table has 9 columns that store various types of data related to the user's diamond consumption, level, and activity. The columns are:

user_id: This is a unique identifier for the user and is encoded as raw data. It is used as the distribution key, meaning the data is distributed across nodes based on this column's value for efficient querying.

from_server: This column records the server from which the user is accessing the application. It is encoded using az64.

simple_type: This column records the type of diamond activity, such as purchase or redemption. It is encoded using lzo.

consume_amt_total: This column records the total amount of diamonds consumed by the user and is stored as a numeric value with two decimal places. It is encoded using az64.

diamond_total_level: This column records the user's diamond level, and it is encoded using az64.

diamond_4_a_cnt: This column records the number of diamonds consumed for activity type "A." It is encoded using az64.

diamond_4_s_cnt: This column records the number of diamonds consumed for activity type "S." It is encoded using az64.

op_time: This column records the time of the diamond activity in character format. It is encoded using lzo.

load_date: This column records the date on which the data was loaded into the table, encoded using az64.

The table's distribution style is set to KEY, and the sort key is set to user_id. This means that the table is distributed based on the user_id column's value, and the data within each node is sorted based on the user_id column's value for efficient querying. Overall, the "app_diamond_report" table can be used to analyze user behavior and consumption patterns within the application, enabling businesses to make informed decisions on their product strategy and pricing.
```







