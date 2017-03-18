+++
date="2015-09-28T16:11:58+05:30"
draft = false
title = "A Redshift UDF to find AB test significance"

+++

I use Amazon's Redshift every day. It's an amazing database for data warehousing and analytics and allows you analyze huge datasets in a blazingly efficient manner using SQL.  

The reason why Redshift is so fast for analysis work is that unlike many other SQL databases, it uses <a href="/web/20161021061459/https://en.wikipedia.org/wiki/Column-oriented_DBMS">columnar storage</a> and is highly optimized for distributing workloads across a cluster of instances.

Redshift is based on PostgreSQL 8.0.2., so it's pretty familiar to anyone who's used Postres or any other mainstream SQL dialect before. Though there is one feature of Postgres that Redshift didn't have until very recently, <a href="/web/20161021061459/http://www.postgresql.org/docs/8.3/static/xfunc.html">User Defined Functions</a> or UDFs for short.

UDF's are really great for encapsulating common logic and also let's you use a more expressive language to implement logic in when SQL isn't quite cutting it.

Amazon did however very recently announce an awesome new feature, <a href="/web/20161021061459/https://aws.amazon.com/blogs/aws/user-defined-functions-for-amazon-redshift/">UDF's that can be implemented in Python</a>. Once you create an embedded function  in Redshift you can use it in any SQL query in the same manner as any native built in function, and Redshift will take care of the input/output bridge between Python and SQL and running your code in a distributed manner on your cluster.

There has been some really great posts about Python UDFs already, by <a href="/web/20161021061459/https://blogs.aws.amazon.com/bigdata/post/Tx1IHV1G67CY53T/Introduction-to-Python-UDFs-in-Amazon-Redshift">Amazon</a>, <a href="/web/20161021061459/http://www.looker.com/blog/amazon-redshift-user-defined-functions">Looker</a> and <a href="/web/20161021061459/https://www.periscope.io/blog/redshift-user-defined-functions-python.html">Periscope</a> and I highly recommend having a look at those if you're curious.

Periscope has also open sourced <a href="/web/20161021061459/https://github.com/PeriscopeData/redshift-udfs">a suite of useful UDF's</a> you can readily use in your own cluster, that comes with a UDF harness you can use to manage and test UDFs

I've been itching to try out writing a UDF to learn more, and with Periscope's harness this turned out to be pretty straightforward.

At Buffer we are constantly running experiments, mostly in the form of AB tests. All of the tracking data around the experiments live in Redshift, and with Looker it's easy for us to model, filter and aggregate the data using LookML and SQL.

One crucial requirement for AB testing to check <a href="/web/20161021061459/https://en.wikipedia.org/wiki/Statistical_significance">the significance</a> of results.

Usually an experiment will have a control and experiment group and for each group we'll have a number of conversions. What we're looking for is a statistical significant difference in the conversions for either group.

There are a number of free online tools that let you easily work out how significant your results are, but I thought it would be great for us to be able to do this right in the database and in Looker.

One of the cool things about the new Python UDF's in Redshift is that they already ship with a bunch of libraries that are often used in data science and analytics work, such as numpy, scipy and pandas.

All this meant writing a UDF to test the null-hypothesis using the <a href="/web/20161021061459/https://en.wikipedia.org/wiki/P-value">p-value</a> was pretty easy to write.

Here is the result:

```sql
create or replace function experiment_result_p_value(control_size float, control_conversion float, experiment_size float, experiment_conversion float)
returns float

stable
as $$
from scipy.stats import chi2_contingency
from numpy import array
observed = array([
  [control_size - control_conversion, control_conversion],
     [experiment_size - experiment_conversion, experiment_conversion]
])
result = chi2_contingency(observed, correction=True)
chisq, p = result[:2]
return p
$$ language plpythonu;
```

This function uses scipy to most of the heavy lifting. All you need to pass in is the size of the control and experiment groups as well as their corresponding conversion numbers and the function will return a <a href="/web/20161021061459/https://en.wikipedia.org/wiki/P-value">p-value</a>.

If the p-value is less than 0.05 you can reject the null-hypothesis and say there is a significant difference between the conversion rates of the the two groups.
