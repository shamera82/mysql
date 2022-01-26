awk is a truly awesome tool, pun intended. you should learn it as it is one of the easiest scripting languages for text manipulation there is.

Here is an awk script named 'createtable.awk' that gives you a good start. you have to save the first line of the csv file to 'somefile.csv' and ONLY the first line

Minor editing is required, you will still have to specify a primary key and varchar(60) is assumed for every field so you can modify the field type to suite...

awk -f createtable.awk somefile.csv to view the output

awk -f createtable.awk somefile.csv > createtable.txt to send to a file then copy/paste into an SQL command

createtable.awk 
```sh
BEGIN{
  FS=","
  tablename = "MyTable"
  printf("\n DROP TABLE %s IF EXISTS",tablename)
  printf("\n CREATE TABLE %s ",tablename)
  printf("\n( ")
}
{
  #printf("\n NF=%d",NF)
  for (i=0; i<=NF; i++)
  {
    printf("\n %s varchar(60) NOT NULL, ",$(i))
  }
}
END{
  printf("\n ); ")
}
```
