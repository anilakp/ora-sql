### Repository for oracle sql-related stuff

# How to use login.sql

1. Place it in your home directory
2. To enable login.sql for every sqlplus session, I did the trick by creating shell alias:
~~~
alias sqlcon='cd ~ && rlwrap sqlplus user/pwd@host:post/instance; cd -'
~~~
NOTE: [https://github.com/hanslub42/rlwrap](rlwrap) is a readline wrapper, useful when combining
with sqlplus cli, enabling you with many possibilities - must have for me, which is navigation through arrows: over the history
