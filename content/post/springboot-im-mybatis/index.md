---
title: 使用 MyBatis 实现即时通讯消息的数据库操作
description: 使用 MyBatis 实现即时通讯消息的数据库操作
date: 2025-03-05
timezone: UTC+8
image: MyBatis.png
slug: springboot-im-mybatis
categories:
  - Java
  - Spring Boot
  - ORM
  - MyBatis
---

# 使用 MyBatis 实现即时通讯消息的数据库操作

## 1.数据库设计

新建一个数据库`db_im`。设计数据表，代码如下：

```sql
# 用户表
CREATE TABLE tb_users (
    user_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    user_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL
);

# 消息表
CREATE TABLE tb_messages (
    message_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    content TEXT NOT NULL,
    state ENUM('sent', 'delivered', 'read') NOT NULL DEFAULT 'sent',
    sender_user_id BIGINT NOT NULL,
    receiver_user_id BIGINT NOT NULL,

    FOREIGN KEY (sender_user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (receiver_user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE
);

# 好友表
CREATE TABLE tb_friends (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    friend_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'accepted', 'blocked') NOT NULL DEFAULT 'pending',

    FOREIGN KEY (user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (friend_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,

    UNIQUE KEY unique_friendship (user_id, friend_id) 
);

# 黑名单表
CREATE TABLE tb_blacklist (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    blocked_user_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (blocked_user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,

    UNIQUE KEY unique_blacklist (user_id, blocked_user_id)
);

# 群聊表
CREATE TABLE tb_groups (
    group_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    group_name VARCHAR(255) NOT NULL,
    owner_user_id BIGINT NOT NULL,

    FOREIGN KEY (owner_user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE
);

# 群聊成员表
CREATE TABLE tb_group_members (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    group_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    joined_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    role ENUM('member', 'admin', 'owner') NOT NULL DEFAULT 'member',

    FOREIGN KEY (group_id) REFERENCES tb_groups(group_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,

    UNIQUE KEY unique_group_membership (group_id, user_id)
);

# 群聊消息表
CREATE TABLE tb_group_messages (
    message_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    content TEXT NOT NULL,
    sender_user_id BIGINT NOT NULL,
    group_id BIGINT NOT NULL,

    FOREIGN KEY (sender_user_id) REFERENCES tb_users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (group_id) REFERENCES tb_groups(group_id) ON DELETE CASCADE
);


# 群读取消息偏移量表
CREATE TABLE tb_group_device_read_offset (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    group_id BIGINT NOT NULL,    
    user_id BIGINT NOT NULL,     
    device_id VARCHAR(255) NOT NULL,  
    last_read_msg_id BIGINT NOT NULL DEFAULT 0,  
    last_read_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,  
    UNIQUE KEY unique_device_read_offset (group_id, user_id, device_id)  
);

# 单聊读取消息偏移量表
CREATE TABLE tb_device_read_offset (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,   
    user_id BIGINT NOT NULL, 
    friend_user_id BIGINT NOT NULL,    
    device_id VARCHAR(255) NOT NULL,  
    last_read_msg_id BIGINT NOT NULL DEFAULT 0,  
    last_read_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,  
    UNIQUE KEY unique_device_read_offset (friend_user_id, user_id, device_id)  
);
```

### 表设计需要注意的点：
1. 由于用户读取消息后不必重复读取消息，所以需要记录用户读取消息的偏移量，即最后一次读取的消息ID。
2. 用户在一个设备上读取了消息，另一个设备上应该有此消息，所以需要记录用户在不同设备上的读取消息偏移量。
3. 为了防止重复添加好友，需要在好友表中添加一个唯一索引。
4. 为了防止重复添加黑名单，需要在黑名单表中添加一个唯一索引。
5. 为了防止重复加入群聊，需要在群聊成员表中添加一个唯一索引。
6. 为了防止重复发送消息，需要在消息表中添加一个唯一索引。
7. 为了防止重复发送群聊消息，需要在群聊消息表中添加一个唯一索引。
8. 为了防止重复读取消息，需要在单聊读取消息偏移量表中添加一个唯一索引。
9. 为了防止重复读取消息，需要在群读取消息偏移量表中添加一个唯一索引。
10. 为了防止重复创建群聊，需要在群聊表中添加一个唯一索引。
11. 为了防止重复创建用户，需要在用户表中添加一个唯一索引。
12. 为了防止偏移量表中重复记录用户读取消息的偏移量，需要在单聊读取消息偏移量表中添加一个唯一索引。并且当重复添加时，更新最后读取消息的时间以及消息ID。-
13. 为了防止偏移量表中重复记录用户读取消息的偏移量，需要在群读取消息偏移量表中添加一个唯一索引。并且当重复添加时，更新最后读取消息的时间以及消息ID。

## 2.实体类设计

创建一个`pojo`包以及 `User`、`Message`、`Friend`、`Blacklist`、`Group`、`GroupMember`、`GroupMessage`、`DeviceReadOffsetMapper`、`GroupDeviceReadOffsetMapper`类。

### 2.1 User 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class User {
    private Long userId;
    private Timestamp createdAt;
    private String userName;
    private String email;
    private String passwordHash;

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPasswordHash() {
        return passwordHash;
    }

    public void setPasswordHash(String passwordHash) {
        this.passwordHash = passwordHash;
    }

    @Override
    public String toString() {
        return "Users{" +
                "userId=" + userId +
                ", createdAt=" + createdAt +
                ", userName='" + userName + '\'' +
                ", email='" + email + '\'' +
                ", passwordHash='" + passwordHash + '\'' +
                '}';
    }
}
```

### 2.2 Message 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;


public class Message {
    public enum State {
        sent, delivered, read
    }
    private Long messageId;
    private Timestamp createdAt;
    private String content;

    private Long senderUserId;

    private Long receiverUserId;

    public Long getSenderUserId() {
        return senderUserId;
    }

    public void setSenderUserId(Long senderUserId) {
        this.senderUserId = senderUserId;
    }

    public Long getReceiverUserId() {
        return receiverUserId;
    }

    public void setReceiverUserId(Long receiverUserId) {
        this.receiverUserId = receiverUserId;
    }

    private State state;

    private User sender;

    private User receiver;

    public void setMessageId(Long messageId) {
        this.messageId = messageId;
    }

    public Long getMessageId() {
        return messageId;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public State getState() {
        return state;
    }

    public void setState(State state) {
        this.state = state;
    }

    public User getSender() {
        return sender;
    }

    public void setSender(User sender) {
        this.sender = sender;
    }

    public User getReceiver() {
        return receiver;
    }

    public void setReceiver(User receiver) {
        this.receiver = receiver;
    }


}
```
数据表里面并没有`State`字段，这里是为了方便代码的编写，所以添加了一个`State`枚举类。也没有`sender`和`receiver`字段，这里添加了`sender`和`receiver`两个对象，用于存储发送者和接收者的信息。后续在查询数据表时，使用左连接来查询这些数据。

### 2.3 Friend 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;


public class Friend {
    public enum Status {
        pending,accepted,rejected
    }
    private Long id;

    private User friend;

    private User user;

    public Long getFriendId() {
        return friendId;
    }

    public void setFriendId(Long friendId) {
        this.friendId = friendId;
    }

    private Long userId;

    private Long friendId;

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    private Timestamp createdAt;

    private Status status;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public User getFriend() {
        return friend;
    }

    public void setFriend(User friend) {
        this.friend = friend;
    }

    public User getUser() {
        return user;
    }

    public void setUser(User user) {
        this.user = user;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }

    public Status getStatus() {
        return status;
    }

    public void setStatus(Status status) {
        this.status = status;
    }
}
```

### 2.4 Blacklist 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class Blacklist {
    private Long id;
    private Long userId;
    private Long blockedUserId;

    private User user;
    private User blockedUser;

    private Timestamp createdAt;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public Long getBlockedUserId() {
        return blockedUserId;
    }

    public void setBlockedUserId(Long blockedUserId) {
        this.blockedUserId = blockedUserId;
    }

    public User getUser() {
        return user;
    }

    public void setUser(User user) {
        this.user = user;
    }

    public User getBlockedUser() {
        return blockedUser;
    }

    public void setBlockedUser(User blockedUser) {
        this.blockedUser = blockedUser;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }
}
```

### 2.5 Group 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class Group {
    private Long groupId;
    private Timestamp createdAt;
    private String groupName;

    public Long getOwnerUserId() {
        return ownerUserId;
    }

    public void setOwnerUserId(Long ownerUserId) {
        this.ownerUserId = ownerUserId;
    }

    private Long ownerUserId;
    private User ownerUser;

    public Long getGroupId() {
        return groupId;
    }

    public void setGroupId(Long groupId) {
        this.groupId = groupId;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }

    public String getGroupName() {
        return groupName;
    }

    public void setGroupName(String groupName) {
        this.groupName = groupName;
    }

    public User getOwnerUser() {
        return ownerUser;
    }

    public void setOwnerUser(User ownerUser) {
        this.ownerUser = ownerUser;
    }
}
```

### 2.6 GroupMember 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;



public class GroupMember {
    public enum Role{
        member,
        admin,
        owner
    }
    private Long id;
    private Group group;
    private Long groupId;
    private User user;
    private Long userId;

    private String remark;  //加个备注
    private Timestamp joinedAt;
    private Role role;

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public Long getGroupId() {
        return groupId;
    }

    public void setGroupId(Long groupId) {
        this.groupId = groupId;
    }

    public String getRemark() {
        return remark;
    }

    public void setRemark(String remark) {
        this.remark = remark;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Group getGroup() {
        return group;
    }

    public void setGroup(Group group) {
        this.group = group;
    }

    public User getUser() {
        return user;
    }

    public void setUser(User user) {
        this.user = user;
    }

    public Timestamp getJoinedAt() {
        return joinedAt;
    }

    public void setJoinedAt(Timestamp joinedAt) {
        this.joinedAt = joinedAt;
    }

    public Role getRole() {
        return role;
    }

    public void setRole(Role role) {
        this.role = role;
    }
}

```

### 2.7 GroupMessage 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class GroupMessage {
    private Long messageId;
    private Timestamp createdAt;
    private String content;
    private Long senderUserId;
    private Long groupId;
    private User senderUser;
    private Group group;

    public Long getSenderUserId() {
        return senderUserId;
    }

    public void setSenderUserId(Long senderUserId) {
        this.senderUserId = senderUserId;
    }

    public Long getGroupId() {
        return groupId;
    }

    public void setGroupId(Long groupId) {
        this.groupId = groupId;
    }

    public Long getMessageId() {
        return messageId;
    }

    public void setMessageId(Long messageId) {
        this.messageId = messageId;
    }

    public Timestamp getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Timestamp createdAt) {
        this.createdAt = createdAt;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public User getSenderUser() {
        return senderUser;
    }

    public void setSenderUser(User senderUser) {
        this.senderUser = senderUser;
    }

    public Group getGroup() {
        return group;
    }

    public void setGroup(Group group) {
        this.group = group;
    }
}
```

### 2.8 DeviceReadOffsetMapper 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class DeviceReadOffset {
    private Long id;
    private Long userId;
    private Long friendUserId;
    private String deviceId;
    private Long lastReadMsgId;
    private Timestamp lastReadTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public Long getFriendUserId() {
        return friendUserId;
    }

    public void setFriendUserId(Long friendUserId) {
        this.friendUserId = friendUserId;
    }

    public String getDeviceId() {
        return deviceId;
    }

    public void setDeviceId(String deviceId) {
        this.deviceId = deviceId;
    }

    public Long getLastReadMsgId() {
        return lastReadMsgId;
    }

    public void setLastReadMsgId(Long lastReadMsgId) {
        this.lastReadMsgId = lastReadMsgId;
    }

    public Timestamp getLastReadTime() {
        return lastReadTime;
    }

    public void setLastReadTime(Timestamp lastReadTime) {
        this.lastReadTime = lastReadTime;
    }
}

```

### 2.9 GroupDeviceReadOffsetMapper 类
```java
package cn.programcx.im.pojo;

import java.sql.Timestamp;

public class GroupDeviceReadOffset {
    private Long id;
    private Long groupId;
    private String deviceId;
    private Long userId;
    private Long lastReadMsgId;
    private Timestamp lastReadTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getGroupId() {
        return groupId;
    }

    public void setGroupId(Long groupId) {
        this.groupId = groupId;
    }

    public String getDeviceId() {
        return deviceId;
    }

    public void setDeviceId(String deviceId) {
        this.deviceId = deviceId;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public Long getLastReadMsgId() {
        return lastReadMsgId;
    }

    public void setLastReadMsgId(Long lastReadMsgId) {
        this.lastReadMsgId = lastReadMsgId;
    }

    public Timestamp getLastReadTime() {
        return lastReadTime;
    }

    public void setLastReadTime(Timestamp lastReadTime) {
        this.lastReadTime = lastReadTime;
    }
}
```

## 3.Mapper 接口设计
为了将封装好的MyBatis的接口和实体类进行绑定，提供操作数据库的方法，需要创建一个`mapper`包，然后在`mapper`包下创建`UserMapper`、`MessageMapper`、`FriendMapper`、`BlacklistMapper`、`GroupMapper`、`GroupMemberMapper`、`GroupMessageMapper`、`DeviceReadOffsetMapper`、`GroupDeviceReadOffsetMapper`接口。

### 3.1 UserMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.User;
import org.apache.ibatis.annotations.Mapper;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface UserMapper {
    void addUser(User user);

    User getUserById(Long id);

    List<User> getUserByName(String name);

    void updateUser(User user);

    void deleteUserById(Long id);
}
```

### 3.2 MessageMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.Message;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface MessageMapper {
    void insertMessage(Message message);
    List<Message> getMessageById(Long id);
    List<Message> getMessageBySenderId(Long id);
    List<Message> getMessageByReceiverId(Long id);
    List<Message> getMessageBySenderAndReceiverId(@Param("senderId") Long senderId, @Param("receiverId") Long receiverId, @Param("lastMessageId") Long lastMessageId);
    List<Message> getMessageBySenderOrReceiverId(Long id);
    void updateMessageState(Message message);
    void deleteMessageById(Long id);
}
```

### 3.3 FriendMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.Friend;
import cn.programcx.im.pojo.Message;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface FriendMapper {
    void addFriend(Friend friend);
    List<Friend> getAllFriends(Friend friend);
    List<Friend> getFriendByState(@Param("userId")Long userId , @Param("state") Friend.Status state);
    void updateFriendStatus(Friend friend);
    void deleteFriend(Friend friend);

}

```

### 3.4 BlacklistMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.Blacklist;
import cn.programcx.im.pojo.User;
import org.apache.ibatis.annotations.Mapper;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface BlacklistMapper {

    void insertBlacklist(Blacklist blacklist);
    List<Blacklist> getBlockedUsers(User user);
    void deleteBlockedUser(Blacklist blacklist);
}
```

### 3.5 GroupMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.Group;
import org.apache.ibatis.annotations.Mapper;
import org.springframework.stereotype.Repository;

import java.util.List;
@Repository
@Mapper
public interface GroupMapper {
   void insertGroup(Group group);
   void deleteGroupById(Long groupId);
   void updateGroupName(Group group);
   Group getGroupById(Long groupId);
   List<Group> getCreatedUserGroups(Long userId);
   List<Group> getAdminUserGroups(Long userId);
   List<Group> getMemberUserGroups(Long userId);
}
```

### 3.6 GroupMemberMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.GroupMember;
import org.apache.ibatis.annotations.Mapper;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface GroupMemberMapper {
    List<GroupMember> getGroupMembersByGroupId(Long groupId);
    void insertGroupMember(GroupMember groupMember);
    void modifyGroupMemberRole(GroupMember groupMember);
    void modifyGroupMemberRemark(GroupMember groupMember);
    void deleteGroupMember(GroupMember groupMember);
}
```

### 3.7 GroupMessageMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.GroupMessage;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@Mapper
public interface GroupMessageMapper {
    void insertGroupMessage(GroupMessage groupMessage);
    void deleteGroupMessageByMessageId(Long messageId);
    List<GroupMessage> getGroupMessagesByGroupId(@Param("groupId") Long groupId,@Param("lastMessageId") Long lastMessageId);
}
```

### 3.8 DeviceReadOffsetMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.DeviceReadOffset;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

@Repository
@Mapper
public interface DeviceReadOffsetMapper {
    void insertDeviceReadOffset(DeviceReadOffset deviceReadOffset);
    DeviceReadOffset getDeviceReadOffset(@Param("userId")Long userId,@Param("friendUserId") Long friendUserId,@Param("deviceId") String deviceId);
}
```

### 3.9 GroupDeviceReadOffsetMapper 接口
```java
package cn.programcx.im.dao;

import cn.programcx.im.pojo.GroupDeviceReadOffset;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

@Repository
@Mapper
public interface GroupDeviceReadOffsetMapper {
    void insertGroupDeviceReadOffset(GroupDeviceReadOffset groupDeviceReadOffset);
    GroupDeviceReadOffset getGroupDeviceReadOffset(@Param("groupId") Long groupId, @Param("deviceId") String deviceId, @Param("userId") Long userId);
}
```
一般，需要简单查询，我们传入id作为参数，如果较为复杂，可能需要传入多个参数，这里使用`@Param`注解来传入多个参数。也可以通过传入`POJO`对象来传入多个参数。

## 4. MyBatis的XML文件设计
在`resources`目录下创建`mapper`目录，然后在`mapper`目录下创建`UserMapper.xml`、`MessageMapper.xml`、`FriendMapper.xml`、`BlacklistMapper.xml`、`GroupMapper.xml`、`GroupMemberMapper.xml`、`GroupMessageMapper.xml`、`DeviceReadOffsetMapper.xml`、`GroupDeviceReadOffsetMapper.xml`文件。

### 4.1 UserMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.UserMapper">
    <insert id="addUser" parameterType="cn.programcx.im.pojo.User" useGeneratedKeys="true" keyProperty="userId">
        insert ignore into tb_users (user_name, email, password_hash)
        values (#{userName},#{email},#{passwordHash})
    </insert>

    <select id="getUserById" parameterType="Long" resultType="cn.programcx.im.pojo.User">
        select * from tb_users where user_id = #{userId}
    </select>

    <select id="getUserByName" parameterType="String" resultType="cn.programcx.im.pojo.User">
        select * from tb_users where user_name = #{userName}
    </select>

    <update id="updateUser" parameterType="cn.programcx.im.pojo.User">
        update tb_users set user_name = #{userName}, email = #{email}, password_hash = #{passwordHash}
        where user_id = #{userId}
    </update>

    <delete id="deleteUserById" parameterType="Long">
        delete from tb_users where user_id = #{userId}
    </delete>
</mapper>
```

### 4.2 MessageMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.MessageMapper">
    <insert id="insertMessage" parameterType="cn.programcx.im.pojo.Message" useGeneratedKeys="true" keyProperty="messageId">
        insert into tb_messages(content, state, sender_user_id, receiver_user_id)
        values (#{content}, #{state}, #{sender.userId}, #{receiver.userId})
    </insert>
    
    <select id="getMessageById" parameterType="Long" resultMap="MessageResultMap">
        select
            m.message_id, m.content, m.created_at, m.state,
            u1.user_id as sender_user_id, u1.user_name as sender_user_name, u1.email as sender_email, u1.password_hash as sender_password_hash,
            u2.user_id as receiver_user_id, u2.user_name as receiver_user_name, u2.email as receiver_email, u2.password_hash as receiver_password_hash
        from tb_messages m
        left join tb_users u1 on m.sender_user_id = u1.user_id
        left join tb_users u2 on m.receiver_user_id  = u2.user_id
        where m.message_id = #{messageId}
    </select>

    <select id="getMessageBySenderId" parameterType="Long" resultMap="MessageResultMap">
        select
            m.message_id,m.content,m.created_at,m.state,
            u1.user_id as sender_user_id, u1.user_name as sender_user_name, u1.email as sender_email, u1.password_hash as sender_password_hash,
            u2.user_id as receiver_user_id, u2.user_name as receiver_user_name, u2.email as receiver_email, u2.password_hash as receiver_password_hash
        from db_im.tb_messages m
        left join db_im.tb_users u1 on m.sender_user_id = u1.user_id
        left join db_im.tb_users u2 on m.receiver_user_id = u2.user_id
        where m.sender_user_id = #{senderId}
    </select>

    <select id="getMessageByReceiverId" parameterType="Long" resultMap="MessageResultMap">
        select
            m.message_id,m.content,m.created_at,m.state,
            u1.user_id as sender_user_id, u1.user_name as sender_user_name, u1.email as sender_email, u1.password_hash as sender_password_hash,
            u2.user_id as receiver_user_id, u2.user_name as receiver_user_name, u2.email as receiver_email, u2.password_hash as receiver_password_hash
        from db_im.tb_messages m
        left join db_im.tb_users u1 on m.sender_user_id = u1.user_id
        left join db_im.tb_users u2 on m.receiver_user_id = u2.user_id
        where m.receiver_user_id = #{receiverId}
    </select>

    <select id="getMessageBySenderOrReceiverId" parameterType="Long" resultMap="MessageResultMap">
        select
            m.message_id,m.content,m.created_at,m.state,
            u1.user_id as sender_user_id, u1.user_name as sender_user_name, u1.email as sender_email, u1.password_hash as sender_password_hash,
            u2.user_id as receiver_user_id, u2.user_name as receiver_user_name, u2.email as receiver_email, u2.password_hash as receiver_password_hash
        from db_im.tb_messages m
        left join db_im.tb_users u1 on m.sender_user_id = u1.user_id
        left join db_im.tb_users u2 on m.receiver_user_id = u2.user_id
        where m.sender_user_id = #{senderId} or m.receiver_user_id = #{receiverId}
    </select>

    <select id="getMessageBySenderAndReceiverId" parameterType="Long" resultMap="MessageResultMap">
        select
            m.message_id,m.content,m.created_at,m.state,
            u1.user_id as sender_user_id, u1.user_name as sender_user_name, u1.email as sender_email, u1.password_hash as sender_password_hash,
            u2.user_id as receiver_user_id, u2.user_name as receiver_user_name, u2.email as receiver_email, u2.password_hash as receiver_password_hash
        from db_im.tb_messages m
        left join db_im.tb_users u1 on m.sender_user_id = u1.user_id
        left join db_im.tb_users u2 on m.receiver_user_id = u2.user_id
        where ((m.sender_user_id = #{senderId} and m.receiver_user_id = #{receiverId}) or (m.sender_user_id = #{receiverId} and m.receiver_user_id =#{senderId})) and m.message_id > #{lastMessageId}
        order by m.created_at desc
    </select>

    <resultMap id="MessageResultMap" type="cn.programcx.im.pojo.Message">
        <id column="message_id" property="messageId"/>
        <result column="content" property="content"/>
        <result column="created_at" property="createdAt"/>
        <result column="state" property="state"/>
        
        <association property="sender" javaType="cn.programcx.im.pojo.User">
            <id column="sender_user_id" property="userId"/>
            <result column="sender_user_name" property="userName"/>
            <result column="sender_email" property="email"/>
            <result column="sender_password_hash" property="passwordHash"/>
        </association>
        
        <association property="receiver" javaType="cn.programcx.im.pojo.User">
            <id column="receiver_user_id" property="userId"/>
            <result column="receiver_user_name" property="userName"/>
            <result column="receiver_email" property="email"/>
            <result column="receiver_password_hash" property="passwordHash"/>
        </association>
    </resultMap>

    <update id="updateMessageState" parameterType="cn.programcx.im.pojo.Message">
        update tb_messages set state = #{state}
        where message_id = #{messageId}
    </update>

    <delete id="deleteMessageById" parameterType="Long">
        delete from tb_messages where message_id = #{messageId}
    </delete>

</mapper>
```

### 4.3 FriendMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.FriendMapper">
    <insert id="addFriend" parameterType="cn.programcx.im.pojo.Friend" useGeneratedKeys="true" keyProperty="id">
        insert ignore into tb_friends(user_id,friend_id,status)
        values (#{user.userId},#{friend.userId},#{status})
    </insert>

    <select id="getAllFriends" resultMap="FriendsResultMap" parameterType="cn.programcx.im.pojo.User">
        select
            f.id, f.user_id,f.friend_id,f.status,
            tu.user_id as friend_user_id, tu.user_name as friend_user_name, tu.email as friend_email,
            tu1.user_id as user_user_id, tu1.user_name as user_user_name, tu1.email as user_email
        from tb_friends f
        left join tb_users tu on tu.user_id = f.friend_id
        left join tb_users tu1 on tu1.user_id = f.user_id
        where f.user_id = #{user.userId}
        order by tu.user_name
    </select>

    <select id="getFriendByState" resultMap="FriendsResultMap" >
        select
            f.id, f.user_id,f.friend_id,f.status,
            tu.user_id as friend_user_id, tu.user_name as friend_user_name, tu.email as friend_email,
            tu1.user_id as user_user_id, tu1.user_name as user_user_name, tu1.email as user_email
        from tb_friends f
        left join tb_users tu on tu.user_id = f.friend_id
        left join tb_users tu1 on tu1.user_id = f.user_id
        where f.user_id = #{userId} and f.status = #{state}
        order by tu.user_name
    </select>

    <update id="updateFriendStatus" parameterType="cn.programcx.im.pojo.Friend">
        update tb_friends
        set status = #{status}
        where user_id = #{user.userId} and friend_id = #{friend.userId}
    </update>

    <delete id="deleteFriend" parameterType="cn.programcx.im.pojo.Friend">
        delete from tb_friends
        where user_id = #{user.userId} and friend_id = #{friend.userId}
    </delete>

    <resultMap id="FriendsResultMap" type="cn.programcx.im.pojo.Friend">
        <id property="id" column="id"/>
        <result property="userId" column="user_id"/>
        <result property="friendId" column="friend_id"/>
        <result property="status" column="status"/>
        <association property="user" javaType="cn.programcx.im.pojo.User">
            <id property="userId" column="user_user_id"/>
            <result property="userName" column="user_user_name"/>
            <result property="email" column="user_email"/>
        </association>

        <association property="friend" javaType="cn.programcx.im.pojo.User">
            <id property="userId" column="friend_user_id"/>
            <result property="userName" column="friend_user_name"/>
            <result property="email" column="friend_email"/>
        </association>
    </resultMap>
    
</mapper>
```

### 4.4 BlacklistMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.BlacklistMapper">
    <insert id="insertBlacklist" parameterType="cn.programcx.im.pojo.Blacklist" useGeneratedKeys="true" keyProperty="id">
        insert ignore into tb_blacklist(user_id,blocked_user_id)
        values (#{user.userId},#{blockedUser.userId})
    </insert>

    <resultMap id="BlacklistResultMap" type="cn.programcx.im.pojo.Blacklist">
        <id property="id" column="id"/>
        <result property="userId" column="user_id"/>
        <result property="blockedUserId" column="blocked_user_id"/>
        <result property="createdAt" column="created_at"/>
        <association property="user" javaType="cn.programcx.im.pojo.User">
            <id property="userId" column="user_user_id"/>
            <result property="userName" column="user_user_name"/>
            <result property="email" column="user_email"/>
        </association>
        <association property="blockedUser" javaType="cn.programcx.im.pojo.User">
            <id property="userId" column="blocked_user_user_id"/>
            <result property="userName" column="blocked_user_user_name"/>
            <result property="email" column="blocked_user_email"/>
        </association>

    </resultMap>

    <select id="getBlockedUsers" resultMap="BlacklistResultMap" parameterType="cn.programcx.im.pojo.User">
        select
            b.id,b.user_id,b.blocked_user_id,b.created_at,
            tu.user_id as user_user_id,tu.user_name as user_user_name,tu.email as user_email,
            tu1.user_id as blocked_user_user_id,tu1.user_name as blocked_user_user_name,tu1.email as blocked_user_email
        from tb_blacklist b
        left join tb_users tu on tu.user_id = b.user_id
        left join tb_users tu1 on tu1.user_id = b.blocked_user_id
        where b.user_id = #{userId}
    </select>

    <delete id="deleteBlockedUser" parameterType="cn.programcx.im.dao.BlacklistMapper">
        delete from tb_blacklist
        where user_id = #{user.userId} and blocked_user_id = #{blockedUser.userId}
    </delete>

</mapper>
```

### 4.5 GroupMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.GroupMapper">
    <resultMap id="groupMapper" type="cn.programcx.im.pojo.Group">
        <id column="group_id" property="groupId" />
        <result property="groupName" column="group_name" />
        <result property="createdAt" column="created_at" />
        <result property="ownerUserId" column="owner_u" />
        <association property="ownerUser" javaType="cn.programcx.im.pojo.User">
            <id column="user_id" property="userId" />
            <result column="user_name" property="userName" />
            <result column="email" property="email" />
        </association>
    </resultMap>
    
    <select id="getGroupById" resultMap="groupMapper" parameterType="Long">
        select
            g.group_id,g.group_name,g.created_at,g.owner_user_id,
            u.user_id,u.user_name,u.email
        from tb_groups g
        left join tb_users u on g.owner_user_id = u.user_id
        where g.group_id = #{groupId}
    </select>

    <select id="getCreatedUserGroups" parameterType="Long" resultMap="groupMapper">
        select
            g.group_id,g.group_name,g.created_at,g.owner_user_id,
            u.user_id,u.user_name,u.email
        from tb_groups g
        left join tb_users u on g.owner_user_id = u.user_id
        where g.owner_user_id = #{userId}
    </select>

    <select id="getMemberUserGroups" parameterType="Long" resultMap="groupMapper">
        select
            g.*,
            u.user_id,u.user_name,u.email
        from tb_groups g
        inner join tb_group_members gm on g.group_id = gm.group_id
        left join tb_users u on g.owner_user_id = u.user_id
        where gm.user_id = #{userId} and gm.role = 'member'
    </select>

    <select id="getAdminUserGroups" parameterType="Long" resultMap="groupMapper">
        select
            g.*,
            u.user_id,u.user_name,u.email
        from tb_groups g
                 inner join tb_group_members gm on g.group_id = gm.group_id
                 left join tb_users u on g.owner_user_id = u.user_id
        where gm.user_id = #{userId} and gm.role = 'admin'
    </select>

    <update id="updateGroupName" parameterType="cn.programcx.im.pojo.Group">
        update tb_groups
        set group_name = #{groupName}
        where group_id = #{groupId}
    </update>

    <insert id="insertGroup" parameterType="cn.programcx.im.pojo.Group"  useGeneratedKeys="true" keyProperty="groupId">
        insert ignore into tb_groups(group_name,owner_user_id)
        values (#{groupName},#{ownerUser.userId})
    </insert>

    <delete id="deleteGroupById" parameterType="Long">
        delete from tb_groups
        where group_id=#{groupId}
    </delete>
</mapper>
```

### 4.6 GroupMemberMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.GroupMemberMapper">
    <resultMap id="groupMemberResultMapper" type="cn.programcx.im.pojo.GroupMember">
        <id column="id" property="id"/>
        <result column="group_id" property="groupId"/>
        <result column="user_id" property="userId"/>
        <result column="role" property="role"/>
        <result column="joined_at" property="joinedAt"/>
        <result column="remark" property="remark"/>
        <association property="user" javaType="cn.programcx.im.pojo.User">
            <id column="user_id" property="userId"/>
            <result column="user_name" property="userName"/>
            <result column="email" property="email"/>
        </association>
        <association property="group" javaType="cn.programcx.im.pojo.Group">
            <id column="group_id" property="groupId"/>
            <result column="group_name" property="groupName"/>
            <result column="created_at" property="createdAt"/>
            <result column="owner_user_id" property="ownerUserId"/>
        </association>
    </resultMap>

    <select id="getGroupMembersByGroupId" resultMap="groupMemberResultMapper" parameterType="Long">
        select
            gm.*,
            u.user_id,u.user_name,u.email,
            g.group_id,g.group_name,g.created_at,g.owner_user_id
        from tb_group_members gm
        left join tb_users u on gm.user_id = u.user_id
        left join tb_groups g on gm.group_id = g.group_id
        where g.group_id = #{groupId}
    </select>

    <insert id="insertGroupMember" parameterType="cn.programcx.im.pojo.GroupMember" keyProperty="id" useGeneratedKeys="true">
        <if test="remark != null">
            insert ignore into tb_group_members(user_id,role,group_id,remark) values (#{user.userId},#{role},#{group.groupId},#{remark})
        </if>
        <if test="remark == null">
            insert into tb_group_members(user_id,role,group_id) values (#{user.userId},#{role},#{group.groupId})
        </if>
    </insert>

    <update id="modifyGroupMemberRole" parameterType="cn.programcx.im.pojo.GroupMember">
        update tb_group_members
        set role = #{role}
        where group_id = #{group.groupId} and user_id = #{user.userId}
    </update>

    <update id="modifyGroupMemberRemark" parameterType="cn.programcx.im.pojo.GroupMember">
        update tb_group_members
        set remark = #{remark}
        where group_id = #{group.groupId} and user_id = #{user.userId}
    </update>

    <delete id="deleteGroupMember" parameterType="cn.programcx.im.pojo.GroupMember">
        delete from tb_group_members
        where id = #{id} or (user_id = #{user.userId} and group_id = #{group.groupId})
    </delete>

</mapper>
```

### 4.7 GroupMessageMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.GroupMessageMapper">
    <resultMap id="groupMessageMap" type="cn.programcx.im.pojo.GroupMessage">
        <id column="message_id" property="messageId"/>
        <result column="content" property="content"/>
        <result column="created_at" property="createdAt"/>
        <result column="sender_user_id" property="senderUserId"/>
        <result column="group_id" property="groupId"/>
        <association property="senderUser" javaType="cn.programcx.im.pojo.User">
            <id column="user_id" property="userId"/>
            <result column="user_name" property="userName"/>
            <result column="email" property="email"/>
        </association>
        <association property="group" javaType="cn.programcx.im.pojo.Group">
            <id column="group_id" property="groupId"/>
            <result column="group_name" property="groupName"/>
            <result column="created_at" property="createdAt"/>
            <result column="owner_user_id" property="ownerUserId"/>
        </association>
    </resultMap>

    <insert id="insertGroupMessage" parameterType="cn.programcx.im.pojo.GroupMessage" useGeneratedKeys="true" keyProperty="messageId">
        insert into tb_group_messages(content,sender_user_id,group_id)
        values (#{content},#{senderUser.userId},#{group.groupId})
    </insert>
    
    <select id="getGroupMessagesByGroupId" resultMap="groupMessageMap" parameterType="Long">
        select
            gm.*,
            u.user_id,u.user_name,u.email,
            g.group_id,g.group_name,g.created_at,g.owner_user_id
        from tb_group_messages gm
        left join tb_users u on gm.sender_user_id = u.user_id
        left join tb_groups g on gm.group_id = g.group_id
        where gm.group_id = #{groupId} and gm.message_id > #{lastMessageId} order by gm.created_at desc
    </select>
    
    <delete id="deleteGroupMessageByMessageId" parameterType="cn.programcx.im.pojo.GroupMessage">
        delete from tb_group_messages
        where message_id = #{messageId}
    </delete>
</mapper>
```

### 4.8 DeviceReadOffsetMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.DeviceReadOffsetMapper">
    <insert id="insertDeviceReadOffset" parameterType="cn.programcx.im.pojo.DeviceReadOffset">
        INSERT INTO tb_device_read_offset (device_id, user_id,friend_user_id, last_read_msg_id)
        VALUES (#{deviceId}, #{userId}, #{friendUserId}, #{lastReadMsgId})
        ON DUPLICATE KEY UPDATE last_read_msg_id = #{lastReadMsgId}
    </insert>
    <select id="getDeviceReadOffset" resultType="cn.programcx.im.pojo.DeviceReadOffset" >
        select * from tb_device_read_offset where device_id = #{deviceId} and user_id = #{userId} and friend_user_id = #{friendUserId}
    </select>
</mapper>
```

### 4.9 GroupDeviceReadOffsetMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.programcx.im.dao.GroupDeviceReadOffsetMapper">
    <resultMap id="groupDeviceReadOffsetResultMap" type="cn.programcx.im.pojo.GroupDeviceReadOffset">
        <id property="id" column="id"/>
        <result property="groupId" column="group_id"/>
        <result property="deviceId" column="device_id"/>
        <result property="userId" column="user_id"/>
        <result property="lastReadMsgId" column="last_read_msg_id"/>
        <result property="lastReadTime" column="last_read_time"/>
    </resultMap>

    <insert id="insertGroupDeviceReadOffset" parameterType="cn.programcx.im.pojo.GroupDeviceReadOffset">
        INSERT INTO tb_group_device_read_offset (group_id, device_id, user_id, last_read_msg_id)
        VALUES (#{groupId}, #{deviceId}, #{userId}, #{lastReadMsgId})
        ON DUPLICATE KEY UPDATE last_read_msg_id = #{lastReadMsgId}
    </insert>
    
    <select id="getGroupDeviceReadOffset" resultMap="groupDeviceReadOffsetResultMap">
        select * from tb_group_device_read_offset where group_id = #{groupId} and device_id = #{deviceId} and user_id = #{userId}
    </select>

</mapper>
```
#### 部分代码解析
- `<insert>`标签用于插入数据，`parameterType`属性指定传入的参数类型，`useGeneratedKeys`属性指定是否使用自动生成的主键，`keyProperty`属性指定主键的属性名。比如我给用户创建了一个账号，也就是执行了`addUser`方法，这时候就会执行`insert`标签中的SQL语句，将用户的信息插入到数据库中。但是如果要获取用户id给用户呢？难道还要再进行一次查询吗？这时候就可以使用`useGeneratedKeys`属性，将`keyProperty`属性指定为`userId`，MyBatis 会自动将这个数值填到传入的类`user`的`userId`里面。这样就可以获取到插入的用户id了。
- 如果要查询的数据将要填到的`POJO`类里面有关联的类，可以使用`<association>`标签，将关联的类的属性填充到`POJO`类中。写insert语句时需要用左连接`left join`查询关联的表，将这些结果填到关联类的属性里面。
- 我们如果已经使用`UNIQUE`约束来保证数据的唯一性，那么在插入数据时，如果数据已经存在，就不会再插入数据，这时候可以使用`ON DUPLICATE KEY UPDATE`语句(比如上述代码的`insertGroupDeviceReadOffset`，这样，用户读取消息后，后端只需要执行这个`insertGroupDeviceReadOffset`，而不需要先查询这个记录是否存在，然后再执行类似于`updateGroupDeviceReadOffset`的方法)，如果数据已经存在，就会更新数据，如果数据不存在，就会插入数据。这些据情况而定，如果不需要更新数据，可以使用`INSERT IGNORE`，这样即使有重复的键，也不会报错
- 对于怎样获取用户未读取的消息，可以使用`where`语句，例如 GroupDeviceReadOffsetMapper.xml 文件的`where m.message_id > #{lastMessageId}`，这样就可以获取到用户未读取的消息了。

*该文章时作者学习 MyBatis 写的，为了迁移运用所学知识。里面代码可能不是最佳实践，仅供学习参考。欢迎广大读者指出错误。*

项目地址：[https://github.com/ProgramCX/flow-im-backend](https://github.com/ProgramCX/flow-im-backend)

该项目会随着作者学习的技术栈的丰富不断完成其它层的设计，优化已经写的代码。