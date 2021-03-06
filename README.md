#fjorm - Fast Java Object Relationship Mapping

Performance testing of Hibernate (and many other available tools like JDO) suggests that Hibernate applications might be orders of magnitude slower than plain JDBC applications. polepos.org tests suggested in 2013 that Hibernate/MySQL, for example, is around 250% slower than MySQL/JDBC in their complex concurrency test http://polepos.sourceforge.net/results/PolePositionClientServer.pdf .

That is how *fjorm* is born, the author of fjorm have seen several projects which performance suffered deeply from using ORM. After analyzing Hibernate code (and some JDBC patterns) he wanted to make ORM better, especially efficient. 

*fjorm* enables users to use relational databases, fast but simple object relationship mapping, without vendor lock-in (no cloud, but it is cloud ready).

#fjorm features:
  * designed to have less than 10% performance overhead over plain SQL
  * facilitate faster application development and cleaner code
  * no XML
  * simple to configure
  * test friendly (unit, integration, etc.)
  * no new query language (no JPQL, no HQL, no OQL, etc.) 
  * no vendor lock-in (no cloud, but it is cloud ready)
  * using of standard SQL 'where' clause possible
  * CRUD ready, but does not support nested objects
  * designed by former Google software engineer
  * it is stable. It has been used by [Numbeo](http://www.numbeo.com), [Tralev](http://www.tralev.com), [Online-Utility](http://www.online-utility.org), [DecisionCrowd](http://www.decisioncrowd.com) etc.


#Example usages (from Tralev.com)

Please look at [Tralev](http://www.tralev.com) before you read this example to understand what the following code does.

Table image_info represent information about image and table image_vote stores votes from the user about the image:
```
create table image_info(
  id int auto_increment primary key,
  uploader_username varchar(255),
  uploader_website varchar (255),
  uploader_ip_add varchar(64),
  uploader_displayable_name varchar(255),
  lat double,
  lng double,
  popularity_score int default 0,
  popularity_entries int default 0,
  moderator_score int default 0,
  pageviews int default 0,
  went_to_map int default 0,
  width int default 0,
  height int default 0,
  timestmp timestamp default CURRENT_TIMESTAMP
)DEFAULT CHARACTER SET utf8;
CREATE INDEX image_info_idx on image_info(id, uploader_username, uploader_ip_add);

create table image_vote(
  id int auto_increment primary key,
  image_id int,
  vote int,
  username varchar(255),
  month int,
  year int,   
  FOREIGN KEY (image_id) REFERENCES image_info(id)
)DEFAULT CHARACTER SET utf8;
CREATE INDEX image_vote_image_idx on image_vote (image_id);
```

This is in Class for representing image_info in Java:
```
@TableName(table = "image_info")
public class ImageInfo {

  @Id
  @AutoGenerated
  public int id;

  public String uploader_username;
  public String uploader_displayable_name;
  public String uploader_website;
  public String uploader_ip_add;
  public double lat;
  public double lng;
  public int popularity_score;
  public int popularity_entries;
  public int moderator_score;
  public int pageviews;
  public int went_to_map;
  public Timestamp timestmp = new Timestamp(new Date().getTime());
  
  @Transient
  public double distance_meters;

  public int width;
  public int height;

}
```

Table for storing info about images user did like:
```
@TableName(table = "image_vote")
public class ImageVote {
  @Id
  @AutoGenerated
  public int id;

  public int image_id;
  public int vote;
  public String username;
  public int month;
  public int year;
}

```

#Look at these examples to see how code with fjorm is much simpler than JDBC: 

###Creating new image_info  
```
  Dao<ImageInfo> imageInfoDao = Dao.<ImageInfo>getDao(ImageInfo.class, TralevDaoProperties.getInstance());
  newImageInfo = imageInfoDao.create(newImageInfo);
  if (newImageInfo.id > 0) {
    // ... code ommitted
  }
```

###Getting uploads from user
```
  List<ImageInfo> imagesFromUser = imageInfoDao.read(" where uploader_username = ? order by id desc limit 1000", email);
```

###Getting images near given coordinates (lat, lng)
```
  List<ImageInfo> imagesToReturn = imageInfoDao.read("where lat > ? and lat < ? and lng > ? and lng < ? limit 2000", 
       lat - 0.2, lat + 0.2, lng - 0.2, lng + 0.2);
  addDistancesToEachImageInfo(imagesToReturn, lat, lng);
  //smarly filter images according to distance and popularity
  Collections.sort(imagesToReturn, Collections.reverseOrder(new ImageInfoComparatorUsingPopularityAndDistance()));
```



###Getting pictures which user liked
```
  List<ImageInfo> imagesUserLiked = imageInfoDao.read(" inner join image_vote on image_info.id = image_vote.image_id " + 
              "where image_vote.username = ? and image_vote.vote = 1 order by image_vote.id desc limit 1000", email);
```

#Cursors - iterating data from the table without loading them all

##Introduction

Cursors are the way to enable application to use data from the select query without loading all applicable rows in the memory.

Fjorm supports cursors with cursor(...) methods in Dao objects.


##Example

This example iterates votes of images on Tralev.com website without loading them all in the memory. It prints the number of positive votes and negative votes in `image_vote` table:

```
      Dao<ImageVote> imageVoteDao = Dao.<ImageVote>getDao(ImageVote.class, TralevDaoProperties.getInstance());
      int positiveVotes = 0;
      int negativeVotes = 0;
      Iterator<ImageVote> imageVoteCursorAllTable = imageVoteDao.cursor("" /* you can put your where clause here */);
      while (imageVoteCursorAllTable.hasNext()) {
        ImageVote imageVote = imageVoteCursorAllTable.next();
        if (imageVote.vote > 0) {
          positiveVotes++;
        } else if (imageVote.vote < 0) {
          negativeVotes++;
        }
      }
      System.out.println("Positive votes = " + positiveVotes + ", negative votes = " + negativeVotes);
```

`Cursor` method supports the same parameters as `read` method and returns `iterator` for data which will be fetched one by one.

For example, to iterate only positive votes in `image_vote` table:
```
      Iterator<ImageVote> imageVoteCursorAllTable = imageVoteDao.cursor("vote > 0");
```

#Configuration of the database and note about testing

##Introduction

To instantiate Dao object, you have to pass a object which extends `DaoProperties` i.e.

```
Dao<ImageInfo> imageInfoDao = Dao.<ImageInfo>getDao(ImageInfo.class, TralevDaoProperties.getInstance());
```


Each class which extends `DaoProperties` has to implement three methods:
```
public abstract String getDbServer();
public abstract String getDbUser();
public abstract String getDbPass();
public abstract String getDriverName();
```

and that's all. Not at all difficult. No XML, you don't need to use java properties if you don't want to (but you an implement it using java properties if you want). Simple.

And this is an example:
```
public class TralevDaoProperties extends DaoProperties {

  static TralevDaoProperties dao = new TralevDaoProperties();
  public static TralevDaoProperties getInstance() {
    return dao;
  }


 static String dbserver= "jdbc:mysql://localhost/tralev?useUnicode=true&characterEncoding=utf-8&autoReconnect=true";
 static String dbuser = "root";
 static String dbpass = "Secret123";
 static String dbDriver = "com.mysql.jdbc.Driver";

  @Override
  public String getDbServer() {
    return dbserver;
  }

  @Override
  public String getDbUser() {
    return dbuser;
  }

  @Override
  public String getDbPass() {
    return dbpass;
  }

  @Override
  public String getDriverName() {
    return dbDriver;
  }

}
```

##Testing

We don't limit you to one particular way how to test applications written using fjorm.

There are few possibilities to consider:
 * you could create different subclasses of `DaoProperties` i.e. `ServerDaoProperties`, `LocalDaoProperties` and `TestDaoProperties` and inject `DaoProperties` with either static call i.e. with method `setDaoProperties` or otherwise
 * you could mock each Dao object (i.e. name it `MockDao`) and inject on aparticular place. You can use [EasyMock](http://easymock.org/) or [Guice](https://github.com/google/guice/wiki/GettingStarted) or other tool for mocking objects
 * you could use a properties file for different database server configurations
 * other way which you do like - we are not trying to limit you here, show some creativity on the way you like to do it
 
#Requirements and dependencies

fjorm is supposed to work on whatever you can run Java onto. 

It has following external dependencies:
  * [Apache Commons Pool](http://commons.apache.org/proper/commons-pool/),   
  * [Apache Commons DBCP](http://commons.apache.org/proper/commons-dbcp/), 

You'll need to use JAR file of your database driver. Fjorm is supposed to work with any database and it has been initially developed using MySQL. You can find a [http://dev.mysql.com/downloads/connector/j/](MySQL driver here).

#CRUD Operations which fjorm enables you to use

For list of supported operations you can look up in Dao.class interface source code.

Some examples (let's learn from examples):

###Example for Create and readFirst
```
  public static boolean writeLikesForUser(int id, int vote, String email) {
    try {
      Dao<ImageVote> imageVoteDao = Dao.<ImageVote>getDao(ImageVote.class, TralevDaoProperties.getInstance());
      ImageVote existingVote = imageVoteDao.readFirst("image_id = ? and username = ?", id, email);
      ImageVote imageVote = new ImageVote();
      if (existingVote != null) {
        imageVote = existingVote;
      }
      imageVote.image_id = id;
      imageVote.month = DateTimeUtils.getMonth();
      imageVote.year = DateTimeUtils.getYear();
      imageVote.username = email;
      imageVote.vote = vote;
      if (existingVote != null) {
        imageVoteDao.update(imageVote);
      } else {
        imageVoteDao.create(imageVote);
      }
      return true;
    } catch (SQLException ex) {
      Logger.getLogger(ImageVoteUtils.class.getName()).log(Level.SEVERE, null, ex);
    }
    return false;
  }

```


##Example for delete and for deleteByKey
```
  public static boolean deleteImage(int id) {
    Logger.getLogger(ImageUtils.class.getName()).log(Level.INFO, "started dequying all...");
    ImageCacheCreatorRunnable.getInstanceAndRunIfNotRunned().dequeueAll(id);
    Logger.getLogger(ImageUtils.class.getName()).log(Level.INFO, "finished dequying all...");
    Dao<ImageVote> imageVoteDao = Dao.<ImageVote>getDao(ImageVote.class, TralevDaoProperties.getInstance());
    try {
      imageVoteDao.delete("image_id = ?", id);
    } catch (SQLException ex) {
      Logger.getLogger(CaptureServlet.class.getName()).log(Level.SEVERE, null, ex);
      return false;
    }
    Dao<ImageInfo> imageInfoDao = Dao.<ImageInfo>getDao(ImageInfo.class, TralevDaoProperties.getInstance());
    String filename = ImageUtils.getOrigFilename(id);
    File file = new File(filename);
    boolean result = true;
    if (file.exists()) {
      result = file.delete();
      deleteNonOriginalCopiesOfImage(id);
    }
    Logger.getLogger(ImageUtils.class.getName()).log(Level.INFO, "finished deleting non original copies...");
    try {
      imageInfoDao.deleteByKey(id);
      Logger.getLogger(ImageUtils.class.getName()).log(Level.INFO, "finished deleting imageInfo by key");
      return result;
    } catch (SQLException ex) {
      Logger.getLogger(ImageUtils.class.getName()).log(Level.SEVERE, null, ex);
      return false;
    }
  }
```

##Examples for read

```
//getting images which are not moderated
imageInfoDao.read("moderator_score = 0 order by id asc limit " + limit);
```

```
// getting images which user did like
imageInfoDao.read(" inner join image_vote on image_info.id = image_vote.image_id " + 
              "where image_vote.username = ? and image_vote.vote = 1 order by image_vote.id desc limit 1000", email);
```

```
//getting images which user did upload
imageInfoDao.read(" where uploader_username = ? order by id desc limit 1000", email);
```

```
//get recent uploads by user. Recent uploads are still not reviewed by moderator
imageInfoDao.read(" where uploader_username = ? and moderator_score = 0 order by id desc limit 100", email);
```


```
///get images near given lat,lng
imageInfoDao.read("where lat > ? and lat < ? and lng > ? and lng < ? limit 2000", lat - 0.2, lat + 0.2, lng - 0.2, lng + 0.2);
```

##Examples for readAll
```
   List<CityInfo> cityInfos = citiesInfoDao.readAll();
```

##Examples for update
```
  public static boolean setModeratorScore(int id, int moderator_score) {
    try {
      Dao<ImageInfo> imageInfoDao = Dao.<ImageInfo>getDao(ImageInfo.class, TralevDaoProperties.getInstance());
      ImageInfo imageInfo = imageInfoDao.readByKey(id);
      if (imageInfo == null) {
        throw new RuntimeException("no image_info for id " + id);
      }
      imageInfo.moderator_score = moderator_score;
      imageInfoDao.update(imageInfo);
      return true;
    } catch (SQLException ex) {
      Logger.getLogger(ImageUtils.class.getName()).log(Level.SEVERE, null, ex);
      return false;
    }
  }
```


```
  public static boolean writeLikesForUser(int id, int vote, String email) {
    try {
      Dao<ImageVote> imageVoteDao = Dao.<ImageVote>getDao(ImageVote.class, TralevDaoProperties.getInstance());
      ImageVote existingVote = imageVoteDao.readFirst("image_id = ? and username = ?", id, email);
      ImageVote imageVote = new ImageVote();
      if (existingVote != null) {
        imageVote = existingVote;
      }
      imageVote.image_id = id;
      imageVote.month = DateTimeUtils.getMonth();
      imageVote.year = DateTimeUtils.getYear();
      imageVote.username = email;
      imageVote.vote = vote;
      if (existingVote != null) {
        imageVoteDao.update(imageVote);
      } else {
        imageVoteDao.create(imageVote);
      }
      return true;
    } catch (SQLException ex) {
      Logger.getLogger(ImageVoteUtils.class.getName()).log(Level.SEVERE, null, ex);
    }
    return false;
  }
```

#Caching strategies

fjorm supports 3 caching strategies for tables:

1. Standard - only caching at the database level

2. `LazyCache` - once the row is read from the database, it will be kept, however complex queries will still go to the database. It keeps cached data in `HashTable` for performance purposes.

3. `FullCache` - on init, the whole content of the database will be read, and standard CRUD operation will use in-memory content. Complex SQL queries still go to the database.

To use `LazyCache` or `FullCache` annotate the Class with its annotations, i.e.
```
@FullCache
@TableName(table = "city")
public class CityInfo {

  @Id
  public int id;

  public String name;
  public String country;
  public double lat;
  public double lng;

  public String toPrintableName() {
    return name + ", " + country;
  }

}

```

#Design rules with app with fjorm


Example Table:
```
create table image_vote(
  id int auto_increment primary key,
  image_id int,
  vote int,
  username varchar(255),
  month int,
  year int,   
  FOREIGN KEY (image_id) REFERENCES image_info(id)
)DEFAULT CHARACTER SET utf8;
CREATE INDEX image_vote_image_idx on image_vote (image_id);
```

Corresponding Mapping:
```
package com.tralev.server;

import fjorm.AutoGenerated;
import fjorm.Id;
import fjorm.TableName;

/**
 * @author mladen
 */
@TableName(table = "image_vote")
public class ImageVote {
  @Id
  @AutoGenerated
  public int id;

  public int image_id;
  public int vote;
  public String username;
  public int month;
  public int year;

}
```

##Rule no1: no submappings - database objects cannot be contained explicitly

m:1 and m:n mappings inside Java Objects are evil and root of many performance problems in existing ORMs for Java. *We don't want to support it.* 

##Rule no2: each field from the database shall be public in Java

Do you really think that `int getVote()` and `void setVote(int vote)` actually is beneficial? If you change the vote 'imageVote.vote = 1;' you know that the field changed, there is side effect. Getters and setters are frequently just overheard unless you work with special cases. We want you to declare fields as public rather than use getters and setters! It produces clearer code without side effects in normal cases.

##Rule no3: field name from the database has to be the same in Java

fjorm requires you to use the same field name in the database as in java. *This is done to prevent confusion*. You shall name fields clear in the database and then fjorm will use that clear name as well. Less confusion. If the database field do not have clear name, rename it, if possible. Otherwise, cope with your database naming decision.

#What you have to know to creating mappings

fjorm do not support automatic generation of mappings. Why? Because we didn't need it. Creating manually Class objects from 'create table' statement usually requires around 1 minute manually and for application with less than 60 tables it is possible to do it in 1 hour. If doing automatic creation, you'd have to set it up and to check if it is done correctly. It might take you approximately same amount of time. That's why with fjorm you have to do it manually.

To map the class to the database use annotation 'TableName', i.e.

```
 @TableName(table = "image_vote")
```

If the field is `Id` annotate it as well, if the field uses the key generated from the database annotate it with `AutoGenerated`, i.e.
```
  @Id
  @AutoGenerated
  public int id;
```


If two or more fields are composite key for the table, specify it with `CompositeKey` annotation, i.e.
```
  @CompositeKey
  public String title;

  @CompositeKey
  public String subtitle;
```

If you want to have a public field in your Class which is not persisted in the database annodate it with `Transient`, i.e. from ImageInfo example
```
  @Transient
  public double distance_meters;
```

##Remember

1. fjorm requires you to use the same field name in the database as in java. *This is done to prevent confusion*. Name fields clear in the database and then fjorm will use that clear name as well. Less confusion.

#Download
[Jar file](https://github.com/mladenadamovic/fjorm/blob/master/dist/fjorm.jar)




