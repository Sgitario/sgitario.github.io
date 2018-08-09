---
layout: post
title: JPA + Hibernate Cascade Scenarios
date: 2016-08-11
tags: [ JPA, Hibernate ]
---
After playing with entity relationships in Hibernate, I wanted to write my notes about the Cascade options and some scenarios for each one. At the end of this post, find a table to match the Cascade Types in Hibernate to JPA and the link to the source code in github.

In the example, we only use a ManyToOne relationship: a role can have many users and an user can have only one role.

```java
package com.sgitario.hibernate.cascade.entities;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.JoinColumn;
import javax.persistence.ManyToOne;

@Entity
public class UserWithNoCascade {
 private Integer id;
 private String firstname;
 private String lastname;

 /** Omit Constructor **/

 @Id
 @GeneratedValue
 public Integer getId() {
  return id;
 }

 /** Omit Setters **/

 @Column(name = "C_FIRSTNAME")
 public String getFirstname() {
  return firstname;
 }

 @Column(name = "C_LASTNAME")
 public String getLastname() {
  return lastname;
 }

 /**
  * Relationship
  */

 private Role role;

 @ManyToOne
 @JoinColumn(name = "role")
 public Role getRole() {
  return role;
 }

 public void setRole(Role role) {
  this.role = role;
 }

}
```

# None Cascade

This is the scenario:
```java
 @Test
 public void scenario() {
  UserWithNoCascade user = new UserWithNoCascade("Dave", "Matthews");
  user.setRole(saveRole(new Role("Manager"))); // 1 save action: 1 insert
  user = saveUser(user); // 1 save action: 1 insert

  user.setFirstname("New Name 2");
  user = saveUser(user); // 1 save action: 2 queries, 1 update
  user.getRole().setRoleName("Employer 2");
  saveRole(user.getRole()); // 1 save action: 1 query, 1 update
 }
```

```
Queries: 
*****: Saving Role
Hibernate: call next value for hibernate_sequence
Hibernate: insert into Role (C_ROLE_NAME, id) values (?, ?)
*****: Saving User
Hibernate: call next value for hibernate_sequence
Hibernate: insert into UserWithNoCascade (C_FIRSTNAME, C_LASTNAME, role, id) values (?, ?, ?, ?)
*****: Saving User #1
Hibernate: select userwithno0_.id as id1_3_0_, userwithno0_.C_FIRSTNAME as C_FIRSTN2_3_0_, userwithno0_.C_LASTNAME as C_LASTNA3_3_0_, userwithno0_.role as role4_3_0_ from UserWithNoCascade userwithno0_ where userwithno0_.id=?
Hibernate: select role0_.id as id1_0_0_, role0_.C_ROLE_NAME as C_ROLE_N2_0_0_ from Role role0_ where role0_.id=?
Hibernate: update UserWithNoCascade set C_FIRSTNAME=?, C_LASTNAME=?, role=? where id=?
*****: Saving Role
Hibernate: select role0_.id as id1_0_0_, role0_.C_ROLE_NAME as C_ROLE_N2_0_0_ from Role role0_ where role0_.id=?
Hibernate: update Role set C_ROLE_NAME=? where id=?
```

With no cascade, you need to save/update the entity and the child entity everytime you make an action. Also, the parent and child entities will be readded to the managed context after a save/update (see #1).

# Persist Cascade

This is the relationship in the entity:

```java
 @ManyToOne(cascade = CascadeType.PERSIST)
 @JoinColumn(name = "role")
 public Role getRole() {
  return role;
 }
```

This is the scenario:

```java
 @Test
 public void scenario() {
  UserWithPersistCascade user = new UserWithPersistCascade("Dave", "Matthews");
  user.setRole(new Role("Manager"));
  user = saveUser(user); // 1 save action: 2 inserts

  user.setFirstname("New Name 1");
  user = saveUser(user); // 1 save action: 2 queries, 1 update
  user.getRole().setRoleName("Employer 1");
  saveRole(user.getRole()); // 1 save action: 1 query, 1 update
 }
```

Queries:
```
*****: Saving User #1
Hibernate: call next value for hibernate_sequence
Hibernate: call next value for hibernate_sequence
Hibernate: insert into Role (C_ROLE_NAME, id) values (?, ?)
Hibernate: insert into UserWithPersistCascade (C_FIRSTNAME, C_LASTNAME, role, id) values (?, ?, ?, ?)
*****: Saving User #2
Hibernate: select userwithpe0_.id as id1_4_0_, userwithpe0_.C_FIRSTNAME as C_FIRSTN2_4_0_, userwithpe0_.C_LASTNAME as C_LASTNA3_4_0_, userwithpe0_.role as role4_4_0_ from UserWithPersistCascade userwithpe0_ where userwithpe0_.id=?
Hibernate: select role0_.id as id1_0_0_, role0_.C_ROLE_NAME as C_ROLE_N2_0_0_ from Role role0_ where role0_.id=?
Hibernate: update UserWithPersistCascade set C_FIRSTNAME=?, C_LASTNAME=?, role=? where id=?
*****: Saving Role #2
Hibernate: select role0_.id as id1_0_0_, role0_.C_ROLE_NAME as C_ROLE_N2_0_0_ from Role role0_ where role0_.id=?
Hibernate: update Role set C_ROLE_NAME=? where id=?
```

With persist cascade, the related entities will be saved/updated when the parent is saved #1, but not updated #2.

# Merge Cascade

This is the relationship in the entity:
```java
 @ManyToOne(cascade = CascadeType.MERGE)
 @JoinColumn(name = "role")
 public Role getRole() {
  return role;
 }
```

This is the scenario:
```java
 @Test
 public void scenario() {
  UserWithMergeCascade user = new UserWithMergeCascade("Dave", "Matthews");
  Role role = saveRole(new Role("Manager")); // 1 save action: 1 insert
  user.setRole(role);
  user = saveUser(user);// 1 save action: 1 insert

  user.setFirstname("New Name 1");
  user.getRole().setRoleName("Employer 1");
  user = saveUser(user);// 1 save action: 1 query, 2 updates
 }
```

Queries:
```
*****: Saving Role
Hibernate: call next value for hibernate_sequence
Hibernate: insert into Role (C_ROLE_NAME, id) values (?, ?)
*****: Saving User
Hibernate: call next value for hibernate_sequence
Hibernate: insert into UserWithMergeCascade (C_FIRSTNAME, C_LASTNAME, role, id) values (?, ?, ?, ?)
*****: Saving User #1
Hibernate: select userwithme0_.id as id1_2_1_, userwithme0_.C_FIRSTNAME as C_FIRSTN2_2_1_, userwithme0_.C_LASTNAME as C_LASTNA3_2_1_, userwithme0_.role as role4_2_1_, role1_.id as id1_0_0_, role1_.C_ROLE_NAME as C_ROLE_N2_0_0_ from UserWithMergeCascade userwithme0_ left outer join Role role1_ on userwithme0_.role=role1_.id where userwithme0_.id=?
Hibernate: update Role set C_ROLE_NAME=? where id=?
Hibernate: update UserWithMergeCascade set C_FIRSTNAME=?, C_LASTNAME=?, role=? where id=?
```

With merge cascade as in no cascade, you need to save every entity individually. Then the child entities will be updated when the parent entity is merged/saved, see #1.

# Remove Cascade

This is the relationship in the entity:
```java
 @ManyToOne(cascade = CascadeType.REMOVE)
 @JoinColumn(name = "role")
 public Role getRole() {
  return role;
 }
```

This is the scenario:
```java
 @Test
 public void scenario() {
  UserWithRemoveCascade user = new UserWithRemoveCascade("Dave", "Matthews");
  Role role = new Role("Manager");
  role = saveRole(role);
  user.setRole(role);
  user = saveUser(user);

  deleteUser(user);
 }
```

Queries:
```
*****: Saving Role
Hibernate: call next value for hibernate_sequence
Hibernate: insert into Role (C_ROLE_NAME, id) values (?, ?)
*****: Saving User
Hibernate: call next value for hibernate_sequence
Hibernate: insert into UserWithRemoveCascade (C_FIRSTNAME, C_LASTNAME, role, id) values (?, ?, ?, ?)
*****: Removing User #1
Hibernate: select userwithre0_.id as id1_4_0_, userwithre0_.C_FIRSTNAME as C_FIRSTN2_4_0_, userwithre0_.C_LASTNAME as C_LASTNA3_4_0_, userwithre0_.role as role4_4_0_ from UserWithRemoveCascade userwithre0_ where userwithre0_.id=?
Hibernate: select role0_.id as id1_0_0_, role0_.C_ROLE_NAME as C_ROLE_N2_0_0_ from Role role0_ where role0_.id=?
Hibernate: delete from UserWithRemoveCascade where id=?
Hibernate: delete from Role where id=?
```

With remove cascade, the child entities will be removed if the parent is deleted, see #1. Bear in mind that the use case of the remove cascade is when removing parent entities. Another scenario is when we remove the relationship (when it's 
a list, doing a list.remove(x)), for this we have the remove_orphan.

```java
 @OneToMany(cascade = CascadeType.REMOVE, orphanRemoval = true)
 @JoinColumn(name = "roles")
 public List getRoles() {
  return roles;
 }
```

# More Cascade options
From JPA:
- ALL: All cascade types in one.
- DETACH: It detaches all child entities if a “manual detach” occurs in parent entity. It implies more queries.
- REFRESH: When the entity manager has references of the parent entity in memory and this parent entity is saved/merged, then all these references and its child relationships will be refreshed (retrieved again from DB).

From Hibernate, the more popular JPA provider, there are some more options:
- SAVE_ACTION: When the Hibernate session is called with saveOrUpdate.
- REPLICATE: When the parent entity state needs to be mirrored between two distinct DataSources
- LOCK: More about this lock and scenarios here.

# Conclusion
Analyze the implications of using one cascade type or another. A wrong cascade type might cause lot of unnecessary queries in database.

Source Code in [GitHub](https://github.com/Sgitario/Java-Hibernate-Examples)
