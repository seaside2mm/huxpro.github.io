# Introduction to SQL  Exercises

Write the following queries in SQL, using the university schema. (We suggest you actually run these queries on a database, using the sample data
that we provide on the Web site of the book, db-book.com. Instructions for
setting up a database, and loading sample data, are provided on the above
Web site.)



a. Find the titles of courses in the Comp. Sci. department that have 3
credits.

```
select title
from course
where dept name = ’Comp. Sci.’
and credits = 3
```



b. Find the IDs of all students who were taught by an instructor named
Einstein; make sure there are no duplicates in the result.

```
select distinct student.ID
from (student join takes using(ID))
join (instructor join teaches using(ID))
using(course id, sec id, semester, year)
where instructor.name = ’Einstein
```

c. Find the highest salary of any instructor.

```

```

d. Find all instructors earning the highest salary (there may be more
than one with the same salary).

```
select ID, name
from instructor
where salary = (select max(salary) from instructor)
```



e. Find the enrollment of each section that was offered in Autumn 2009.

```
select course id, sec id, count(ID)
from section natural join takes
where semester = ’Autumn’
and year = 2009
group by course id, sec id
```



f. Find the maximum enrollment, across all sections, in Autumn 2009.

```
select max(enrollment)
from (select count(ID) as enrollment
from section natural join takes
where semester = ’Autumn’
and year = 2009
group by course id, sec id)
```



g. Find the sections that had the maximum enrollment in Autumn 2009

