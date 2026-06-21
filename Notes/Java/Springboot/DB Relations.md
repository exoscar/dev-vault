### Many-to-Many Relationship

A user can belong to multiple workspaces.
```
Rahul
 ├── Workspace A
 ├── Workspace B
 └── Workspace C
```
A workspace can contain multiple users.
```
Workspace A
 ├── Rahul
 ├── Priya
 └── John
```

Relationship:
`User ↔ Workspace`

Many users 
many workspaces
This is a **Many-to-Many** relationship.

### Join Table
A join table is an intermediate table that stores associations between two entities participating in a many-to-many relationship.

workspace_members conntecting users,workspaces

### JPA ManyToMany
```
@Entity
public class User {

    @Id
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "workspace_members",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "workspace_id")
    )
    private Set<Workspace> workspaces;
}
```

```
@Entity
public class Workspace {

    @Id
    private Long id;

    @ManyToMany(mappedBy = "workspaces")
    private Set<User> members;
}
```

#### Owning side 
the side that defines:
`@JoinTable` is the owning side
eg. Users
#### Inverse Side
the side that uses:
`mappedBy` is the inverse side
eg. Workspace

Why do we need an owning side? -- JPA must know
`Who updates workspace_members table?`
the owning side controls relationship updates.

ManyToMany looks great. only when the mapped data only has two cols. that maps both(ids), what if wanted to add extra fields for workspace_members

this is where manytoman breaks.

### Production reality
most systems evolve into:
```
User
   ↕
WorkspaceMember
   ↕
Workspace
```
Instead of 
`User ↔ Workspace`


## Join Entity Pattern
A design where the join table is represented as a dedicated entity because the relationship itself contains business data.
```
@Entity
public class WorkspaceMember {

    @Id
    private Long id;

    @ManyToOne
    private User user;

    @ManyToOne
    private Workspace workspace;

    private String role;

    private LocalDateTime joinedAt;
}
```

Now WorkspaceMember becomes a first class entity.


Once something becomes a business concept, it deserves its own entity.

### Industry Rule of Thumb
If the join table contains only:
```
user_id
workspace_id
```
ManyToMany is acceptable

The moment you need any extra fields move to join entity.


### Why do many teams avoid `@ManyToMany`?
> @ManyToMany works for simple associations, but real-world relationships often gain additional attributes such as roles, status, timestamps, and audit information. Once the relationship itself contains business data, a dedicated join entity provides better flexibility, maintainability, and domain modeling.


