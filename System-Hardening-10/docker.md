**Question 1:**

Remove a user from the Docker group to enhance security. The user ``develop`` should no longer have Docker privileges.
Use the ``gpasswd`` command to remove the user from the ``docker`` group.

**Step 1 — Check current docker group members**
```
cat /etc/group | grep docker
# Expected: docker:x:XXX:develop (develop is currently in docker group)
```

**Step 2 — Remove develop from docker group**
```
gpasswd -d develop docker
```

**Step 3 — Verify develop is removed**
```
id develop
# develop should NOT show docker in groups
```

