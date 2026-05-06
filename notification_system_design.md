### Stage 2:
My Choice of Database: PostgreSQL
Reason: Better Consistency, Better Indexing support, supports enum, supports partioning even for large tables.

Database Schema:
1. Students Table:
```
CREATE TABLE students (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

```
2. Notifications Table:
```
CREATE TYPE notification_type AS ENUM ('Event', 'Result', 'Placement');

CREATE TABLE notifications (
    id UUID PRIMARY KEY,
    notification_type notification_type NOT NULL,
    title VARCHAR(200) NOT NULL,
    message TEXT NOT NULL,
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
3. Student Notifications Table:
```
CREATE TABLE student_notifications (
    id BIGSERIAL PRIMARY KEY,
    student_id BIGINT NOT NULL REFERENCES students(id),
    notification_id UUID NOT NULL REFERENCES notifications(id),
    is_read BOOLEAN DEFAULT FALSE,
    delivered_at TIMESTAMP,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(student_id, notification_id)
);
```
This design avoids duplicate notifications, supports one to many student notification. Keeps student's read status, scales everything better instead of one large table.

Queries:
1. To get all notifications for a student:
```
SELECT n.id, n.notification_type, n.title, n.message, sn.is_read, n.created_at
FROM student_notifications sn
JOIN notifications n ON n.id = sn.notification_id
WHERE sn.student_id = $1
ORDER BY n.created_at DESC
LIMIT 20;
```
2. To get all unread notifications of a student:
```
SELECT n.id, n.notification_type, n.title, n.message, n.created_at
FROM student_notifications sn
JOIN notifications n ON n.id = sn.notification_id
WHERE sn.student_id = $1
AND sn.is_read = false
ORDER BY n.created_at DESC
LIMIT 20;
```
3. Mark single notification as read:
```
UPDATE student_notifications
SET is_read = true,
    read_at = CURRENT_TIMESTAMP
WHERE student_id = $1
AND notification_id = $2;
```
4. Mark all as read:
```
UPDATE student_notifications
SET is_read = true,
    read_at = CURRENT_TIMESTAMP
WHERE student_id = $1
AND is_read = false;
```
5. Count unread notifications:
```
SELECT COUNT(*) AS unread_count
FROM student_notifications
WHERE student_id = $1
AND is_read = false;
```

Cons when data becomes too large:
As number of students increase, notifications also increase. This may lead to problems like:
- Queries may become slower.
- Tables may become very large.
- Fetching unread notifications may take more time.
- Sending one notification to thousands of students creates many rows.
- Too many page-load requests may put pressure on the database.
- Old data may make searching slower.

To fix these we may:
1. Use indexes
2. Instead of loading entire notifications, load certain amount of notifications using Pagination.
3. Use Redis Cache
4. Archive old notifications
5. use Paritions

### Stage 3:
Given Query:
```
SELECT * FROM notifications
WHERE studentID = 1042 AND isRead = false
ORDER BY createdAt DESC;
```
This query is not fully accurate as we use a different database design in stage 2.

In Stage 2, we separated the data into:
- notifications table for notification content
- student_notifications table for student-specific read/unread status

So this query becomes slow because the system now has:
50,000 students
5,000,000 notifications
Reasons:
- The database may need to scan many rows to find matching records.
- It also has to sort the results using ORDER BY createdAt DESC.
- SELECT * fetches all columns, even if all of them are not needed.
- If there is no proper index, performance becomes poor as data grows.
- So the query is slow because it is working on a very large table and may not be using the best structure or indexing.

So a better query is:
```
SELECT n.id, n.notification_type, n.title, n.message, n.created_at
FROM student_notifications sn
JOIN notifications n ON n.id = sn.notification_id
WHERE sn.student_id = 1042
AND sn.is_read = false
ORDER BY n.created_at DESC;
```
This is better because:
- It follows the correct database design.
- Read/unread status is checked from the correct table.
- It only selects the required columns instead of using SELECT *.
- It is easier to optimize using indexes.

### Stage 4:
The problem is that notifications are being fetched from the database every time a student loads a page. If many students are using the system at the same time, this creates too many database requests and makes the app slow.  

So, we need to reduce the load on the database and improve performance.  

Solution 1: Use Caching  
One good solution is to use a cache system like Redis.
We can store:
- unread notification count
- latest notifications for a student
- Instead of asking the database every time, the app can first check Redis.  
Redis is much faster than the main database. It reduces repeated database queries and it improves response time for the user. But, Cache data may become slightly outdated for a short time.
We need extra logic to update or clear the cache when notifications change.  

Solution 2: Use Pagination
Instead of loading all notifications at once, we should load only a small number, such as 10 or 20 at a time.

Solution 3: Use Real-Time Updates Instead of Refetching on Every Page Load
We can use Server-Sent Events (SSE) or WebSockets to push new notifications to students in real time.

Solution 4: Use Unread Count Separately
The frontend often only needs the number of unread notifications, not the full notification list.

Solution 5: Use Conditional Requests
We can use headers like:

ETag
Last-Modified
If notifications have not changed, the server can return:

304 Not Modified  
It Saves bandwidth, reduces unnecessary full responses, improves performance when data has not changed But, Extra logic is needed on both frontend and backend and the server still needs to check whether data changed.

To improve performance, I would not fetch all notifications from the database on every page load. 

### Stage 5:
The given implementation is not good for 50,000 students because it is slow, unreliable, and does not handle failures properly.
Because, Current implementation follows, sequential processing, doesn't retry, has no queue nor background workers, has no proper status tracking  
If 200 emails fail, we should retry only those failed email jobs.  
And Saving to DB and sending email should not happen together in one blocking loop, because DB operations are internal but email depends on an external service.

A better design is:
- save notifications first
- use queues for email and push delivery
- process them with background workers
- retry failures automatically
- track status clearly