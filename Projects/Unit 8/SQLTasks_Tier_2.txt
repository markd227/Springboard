/* Welcome to the SQL mini project. You will carry out this project partly in
the PHPMyAdmin interface, and partly in Jupyter via a Python connection.

This is Tier 2 of the case study, which means that there'll be less guidance for you about how to setup
your local SQLite connection in PART 2 of the case study. This will make the case study more challenging for you: 
you might need to do some digging, aand revise the Working with Relational Databases in Python chapter in the previous resource.

Otherwise, the questions in the case study are exactly the same as with Tier 1. 

PART 1: PHPMyAdmin
You will complete questions 1-9 below in the PHPMyAdmin interface. 
Log in by pasting the following URL into your browser, and
using the following Username and Password:

URL: https://sql.springboard.com/
Username: student
Password: learn_sql@springboard

The data you need is in the "country_club" database. This database
contains 3 tables:
    i) the "Bookings" table,
    ii) the "Facilities" table, and
    iii) the "Members" table.

In this case study, you'll be asked a series of questions. You can
solve them using the platform, but for the final deliverable,
paste the code for each solution into this script, and upload it
to your GitHub.

Before starting with the questions, feel free to take your time,
exploring the data, and getting acquainted with the 3 tables. */


/* QUESTIONS 
/* Q1: Some of the facilities charge a fee to members, but some do not.
Write a SQL query to produce a list of the names of the facilities that do. */

SELECT name
FROM Facilities
WHERE membercost <> 0.0;


/* Q2: How many facilities do not charge a fee to members? */

SELECT COUNT(facid)
FROM Facilities
WHERE membercost = 0.0;


/* Q3: Write an SQL query to show a list of facilities that charge a fee to members,
where the fee is less than 20% of the facility's monthly maintenance cost.
Return the facid, facility name, member cost, and monthly maintenance of the
facilities in question. */

SELECT facid, name, membercost, monthlymaintenance
FROM Facilities
WHERE membercost <> 0.0 
	AND membercost < 0.2 * monthlymaintenance;


/* Q4: Write an SQL query to retrieve the details of facilities with ID 1 and 5.
Try writing the query without using the OR operator. */

SELECT *
FROM Facilities
WHERE facid IN (1,5);


/* Q5: Produce a list of facilities, with each labelled as
'cheap' or 'expensive', depending on if their monthly maintenance cost is
more than $100. Return the name and monthly maintenance of the facilities
in question. */

SELECT name, monthlymaintenance,
	CASE WHEN monthlymaintenance >= 100
			THEN 'expensive'
		ELSE 'cheap' END AS cost
FROM Facilities;


/* Q6: You'd like to get the first and last name of the last member(s)
who signed up. Try not to use the LIMIT clause for your solution. */

SELECT firstname, surname
FROM Members
WHERE joindate = (SELECT MAX(joindate)
                  FROM Members);


/* Q7: Produce a list of all members who have used a tennis court.
Include in your output the name of the court, and the name of the member
formatted as a single column. Ensure no duplicate data, and order by
the member name. */

SELECT DISTINCT name, 
	CONCAT(firstname, ' ', surname) as member
FROM Facilities
INNER JOIN (
    SELECT facid, memid
	FROM Bookings
	WHERE facid IN (0,1)
    ) AS tennis_bookings
USING (facid)
INNER JOIN Members
USING (memid)
WHERE firstname <> 'GUEST'
ORDER BY member;


/* Q8: Produce a list of bookings on the day of 2012-09-14 which
will cost the member (or guest) more than $30. Remember that guests have
different costs to members (the listed costs are per half-hour 'slot'), and
the guest user's ID is always 0. Include in your output the name of the
facility, the name of the member formatted as a single column, and the cost.
Order by descending cost, and do not use any subqueries. */

SELECT name,
	CONCAT(firstname, ' ', surname) as member,
	CASE WHEN memid = 0 THEN slots * guestcost
		ELSE slots * membercost END AS cost
FROM Bookings
INNER JOIN Facilities
USING(facid)
INNER JOIN Members
USING(memid)
WHERE starttime >= '2012-09-14' AND starttime < '2012-09-15'
	AND CASE WHEN memid = 0 THEN slots * guestcost
		ELSE slots * membercost END > 30
ORDER BY cost DESC;



/* Q9: This time, produce the same result as in Q8, but using a subquery. */

SELECT name,
	(SELECT CONCAT(firstname, ' ', surname)
     	FROM Members WHERE Members.memid = Bookings.memid) AS member,
	CASE WHEN memid = 0 THEN slots * guestcost
		ELSE slots * membercost END AS cost
FROM Bookings
INNER JOIN Facilities
USING(facid)
WHERE starttime >= '2012-09-14' AND starttime < '2012-09-15'
	AND CASE WHEN memid = 0 THEN slots * guestcost
		ELSE slots * membercost END > 30
ORDER BY cost DESC;


/* PART 2: SQLite

Export the country club data from PHPMyAdmin, and connect to a local SQLite instance from Jupyter notebook 
for the following questions.  

QUESTIONS: 
/* Q10: Produce a list of facilities with a total revenue less than 1000.
The output of facility name and total revenue, sorted by revenue. Remember
that there's a different cost for guests and members! */

# Q10 - list of facilities with total revenue less than 1000

# Get list of total revenue by facility
con = engine.connect()
rs = con.execute('''SELECT name, 
                        SUM(CASE WHEN memid = 0 THEN slots * guestcost 
                        ELSE slots * membercost END) AS revenue
                    FROM Bookings
                    INNER JOIN Facilities
                    USING(facid)
                    GROUP BY facid;''')
q10_results = pd.DataFrame(rs.fetchall())
con.close()

# Assign column headers and filter for results
q10_results.columns = rs.keys()
q10_results = q10_results[q10_results['revenue'] < 1000]
q10_results


/* Q11: Produce a report of members and who recommended them in alphabetic surname,firstname order */

# Q11 - report of members and who recommended them

# Get a list of members and who recommended them (by member id)
con = engine.connect()
rs = con.execute('''SELECT t.surname, t.firstname, 
                        CASE WHEN recommendedby = 0 THEN recommendedby
                            ELSE (SELECT t2.firstname || t2.surname
                                  FROM Members AS t2
                                  WHERE t.recommendedby = t2.memid) END AS recommendedby
                    FROM Members AS t
                    ORDER BY surname, firstname;''')
q11_results = pd.DataFrame(rs.fetchall())
con.close()

q11_results.columns = rs.keys()

q11_results


/* Q12: Find the facilities with their usage by member, but not guests */

# Q12 - facilities with their usage by member, ignoring guest usage
con = engine.connect()
rs = con.execute('''SELECT name, 
	firstname || ' ' || surname as member,
	COUNT(memid) as num_bookings
FROM Bookings
INNER JOIN Facilities
USING(facid)
INNER JOIN Members
USING(memid)
WHERE memid <> 0
GROUP BY memid
ORDER BY name, member;''')

q12_results = pd.DataFrame(rs.fetchall())
con.close()

q12_results.columns = rs.keys()

q12_results


/* Q13: Find the facilities usage by month, but not guests */

# Q13 - list facilities with their usage by month, ignoring guest usage
con = engine.connect()
rs = con.execute('''SELECT name,
	SUM(CASE WHEN strftime('%m', starttime) = '01' AND memid <> 0 THEN slots
        	ELSE 0 END) AS january,
    SUM(CASE WHEN strftime('%m', starttime) = '02' AND memid <> 0 THEN slots
        	ELSE 0 END) AS february,
	SUM(CASE WHEN strftime('%m', starttime) = '03' AND memid <> 0 THEN slots
        	ELSE 0 END) AS march,
	SUM(CASE WHEN strftime('%m', starttime) = '04' AND memid <> 0 THEN slots
        	ELSE 0 END) AS april,
	SUM(CASE WHEN strftime('%m', starttime) = '05' AND memid <> 0 THEN slots
        	ELSE 0 END) AS may,
	SUM(CASE WHEN strftime('%m', starttime) = '06' AND memid <> 0 THEN slots
        	ELSE 0 END) AS june,
	SUM(CASE WHEN strftime('%m', starttime) = '07' AND memid <> 0 THEN slots
        	ELSE 0 END) AS july,
	SUM(CASE WHEN strftime('%m', starttime) = '08' AND memid <> 0 THEN slots
        	ELSE 0 END) AS august,
	SUM(CASE WHEN strftime('%m', starttime) = '09' AND memid <> 0 THEN slots
        	ELSE 0 END) AS september,
	SUM(CASE WHEN strftime('%m', starttime) = '10' AND memid <> 0 THEN slots
        	ELSE 0 END) AS october,
	SUM(CASE WHEN strftime('%m', starttime) = '11' AND memid <> 0 THEN slots
        	ELSE 0 END) AS november,
	SUM(CASE WHEN strftime('%m', starttime) = '12' AND memid <> 0 THEN slots
        	ELSE 0 END) AS december,
	SUM(CASE WHEN memid <> 0 THEN slots
        	ELSE 0 END) AS total_mem_bookings
FROM Bookings
INNER JOIN Facilities
USING(facid)
GROUP BY facid;''')
q13_results = pd.DataFrame(rs.fetchall())
con.close()

q13_results.columns = rs.keys()

q13_results











