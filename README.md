Introduction
============

[urlblacklist.com](http://urlblacklist.com/) has a tremendous resource available for people to be able to better secure their networks. It provides a complete list of millions of URLs that have been categorized as things like “porn”, “chat”, and things of the like. This list could then easily be used to block people in a company from accessing social networking sites or preventing employees from doing a job search.

The problem with Dansguardian and Squidguard
--------------------------------------------

When I searched for a way to do easily integrate this with Squid I found [Dansguardian](http://dansguardian.org/) and [Squidguard](http://www.squidguard.org/). These are very powerful and great resources. My goal here is not to knock these two solutions as I support them fully. In fact, in most cases those will be all that you need. So if you just need to be able to block certain websites based on categories and you don't need complicated ACL rules, stop reading here and try either of those resources.

Both Squidguard and Dansguardian become the dominant factor in Squid. Squidguard uses url\_rewrite\_program (an option in Squid) and Dansguardian also becomes a middleman and then passes the request to Squid at the end to do the actual proxying. Since both of these solutions effectively take over and will deny the blacklisted URLs *before* Squid has a chance to run our more complicated ACL rules, they are not good in our situation.

Namely, my problem was that I wanted to use the Squid LDAP transparent proxy authentication script to allow for custom authentication with LDAP/Active Directory and the benefits of urlblacklist.com URLs

The solution
------------

The solution I've come up with is two custom scripts. The first script (**merge\_blacklist**) will download the blacklist gzip file from urlblacklist.com and enter it into a MySQL database. The second (**url\_lookup**) is a second run by Squid to do the actual URL lookups from that database.

Prerequisites
-------------

This article assumes that you have MySQL setup and Squid is already installed with it mostly configured and working.

Installation
============

Downloading and installing
--------------------------

1.  Copy **url\_lookup** into /usr/local/bin and make sure it's executable by at least the Squid user if not by all (chmod +x url\_lookup).
2.  Copy **merge\_blacklist** also into /usr/local/bin and make sure it has the same permissions and can be executed by at least the user who will be running it.

Configuring
-----------

Make sure to modify both **merge\_blacklist** and **url\_lookup** with your proper database info. Otherwise just follow the guide and use what I have here.

**Note that** to “modify” them means to open them up with your favorite editing app and take a look at the lines I have highlighted below for proper modification.

### merge\_blacklist

    my $mysql_host = 'localhost';
    my $mysql_user = 'squidaccess';
    my $mysql_pass = 'squidaccess';
    my $mysql_db = 'squidaccess';

Also note, when testing uncomment the testing line below with “smalltestlist” in the URL. Using the bigblacklist for testing is against urlblacklist.com etiquette.

    #my $blacklist_url = "http://urlblacklist.com/cgi-bin/commercialdownload.pl?type=download&file=bigblacklist";
    my $blacklist_url = "http://urlblacklist.com/cgi-bin/commercialdownload.pl?type=download&file=smalltestlist";

### url\_lookup

    my $mysql_host = 'localhost';
    my $mysql_user = 'squidaccess';
    my $mysql_pass = 'squidaccess';
    my $mysql_db = 'squidaccess';

MySQL database setup
--------------------

Let's set up the MySQL database for use with the script.

    # mysql
    mysql> CREATE DATABASE squidaccess;
    mysql> GRANT ALL PRIVILEGES ON squidaccess.* TO squidauth@localhost IDENTIFIED BY 'squidaccess';
    mysq> quit

You can certainly use another database already in existence if you desire. The **merge\_blacklist** script will automatically create three tables: temp, blacklist, and blacklist\_old. These it keeps in rotation every time it is updated.

Configuring Squid
-----------------

Now make these changes in the **/etc/squid/squid.conf**:

    # Note that where it says chat,porn replace with a comma separated list WITHOUT SPACES
    # of all of the categories you wish to block.
    external_acl_type urlblacklist_lookup ttl=60 %URI /usr/local/bin/url_lookup chat,porn

    # Put these with your ACLs
    acl urlblacklist external urlblacklist_lookup

    # Put this at the appropriate place and change it for your configuration
    http_access allow localnet urlblacklist

**Note**: keep in mind that *localnet* should be either defined with your localnet or replaced with the acl IP src that you want. For example, my *localnet* is defined as:

    acl localnet src 192.168.1.0/24

And it should be defined *above* in your **squid.conf** where you reference it.

Script usage details
--------------------

### url\_lookup

When running the script, you can either provide a comma separated list (without spaces) to the script directly as in

    /usr/local/bin/url_lookup chat,porn,socialnetworking,abortion

Or you can also provide a file containing all of the categories on a single line. For example:

    /usr/local/bin/url_lookup -f /etc/squid/blacklist_categories

And then in **/etc/squid/blacklist\_categories** have:

    chat
    porn
    socialnetworking
    abortion

##### Whitelist mode

Also, providing the *-w* option to the script puts the script in “whitelist” mode. This means that a URL in the categories supplied will not generate a *denied* (ERR) message to Squid but an *allow* (OK). This is useful for whitelisted categories in the blacklist list.

#### Testing

url\_lookup is meant to be used just with Squid. You can run it to test it if you'd like. Just run it normally and it'll bring you to a command prompt. Then type in the URLs and see what works and what does not.

For example:

    # ./url_lookup porn
    http://www.facebook.com
    OK
    http://www.porn.com
    ERR message="URL found in category porn"
    ^C

#### Debugging

You can toggle the debug options to see in real time what queries the script is running and what the auth status is by setting the **$debug** option in the script to 1. You will then see the output in */tmp/receivedurls*.

Getting it running
------------------

Now run the **merge\_blacklist** script. If it finishes successfully and you get no errors restart Squid and watch it work!

Increasing performance
----------------------

Since each URL will need to be looked up in a database of millions of string keys, you want to get the best performance out of your lookups. It's always a good idea then to minimize disk IO.

### Tuning MySQL for an InnoDB table

Since the *blacklist* table uses the InnoDB engine you can tune your MySQL. This is optional, but it will likely increase performance.

In the MySQL **/etc/my.cnf** or **/etc/mysql/my.cnf**, below the *\[mysqld\]* modify the following.

    innodb_buffer_pool_size = 256M
    innodb_log_file_size = 128M
    innodb_log_buffer_size = 32M
    innodb_file_per_table

    # This also may be a good idea because the INSERT statement lines get very long
    max_allowed_packet = 16M

Note that these are my settings. You can set *innodb\_buffer\_pool\_size* to whatever number your memory allows (but keep in mind that MySQL will likely take it up so be wise if you have other things on the server!). Then set *innodb\_log\_file\_size* to half the size of *innodb\_buffer\_pool\_size*.

Now **don't** make these changes just yet and restart MySQL! You will break it. We will have to move out the old ib\_logfiles before we do. So do the following:

1.  Make a full backup of all of your databases using *mysqldump* just in case.
2.  Stop all processes using MySQL (like *apache* if you have web scripts running it, etc).
3.  Open up the MySQL command line and run *set global innodb\_fast\_shutdown=0;* as root.
4.  Stop the MySQL service.
5.  Make sure MySQL made a clean shutdown. Consult some log files.
6.  Make the changes above to the *my.cnf* file.
7.  Move all ib\_logfiles out of the MySQL data directory (*mv /var/lib/mysql/ib\_logfile\* ~* works).
8.  Start MySQL back up.
9.  Check to see the *innodb\_log\_file\_size* is working (*du -hs /var/lib/mysql/ib\_logfile\**).
10. Look for any log file errors and run *mysqlcheck -A* if you want to make sure all tables made it through.
11. Start up all services that use MySQL again.

If anything happens during this process and you experience some major corruptions you can either try moving your ib\_logfiles back or just blowing away the MySQL data directory and restoring your dump.

URLBlacklist.com policy
=======================

Please note that the nice people at [urlblacklist.com](http://urlblacklist.com/) work on the honor system. If you are going to use their service, please subscribe as they ask. This script is not meant to be used to abuse their downloading policy.

References
==========

-   [<http://urlblacklist.com/>](http://urlblacklist.com/)
-   [<http://www.squidguard.org>](http://www.squidguard.org)
-   [<http://dansguardian.org/>](http://dansguardian.org/)
-   [<http://www.zarafa.com/wiki/index.php/MySQL_tuning>](http://www.zarafa.com/wiki/index.php/MySQL_tuning)
