---
tags:
  - Docker
  - Linux
---
Docker commands usually require elevated access because the Docker daemon runs with root-level privileges.

## Do I need to be root?

No, you do **not** need to log in as `root`.

You have two common ways to use Docker:

- Use `sudo` before each Docker command
- Add your user to the `docker` group so you can run Docker without `sudo`

---

## Option 1: Use sudo

Example:

```bash
sudo docker ps
sudo docker run hello-world
```

### Pros
- Simple
- Explicit privilege escalation
- Safer for shared or strict environments

### Cons
- Repetitive for daily use

---

## Option 2: Add user to docker group

This is the most common setup for personal development machines.

### Add current user to docker group
```bash
sudo usermod -aG docker $USER
```

### Apply group change in current shell
```bash
newgrp docker
```
Starts a new shell session with `docker` as the active group, so the current terminal can use Docker commands immediately after adding the user to the `docker` group
Because when you add your user to the `docker` group, the system usually does not apply that new group membership to your **current shell** automatically
### Test it
```bash
docker ps
```

You can also log out and log back in instead of using `newgrp`.

### Pros
- Convenient
- No need to type `sudo` every time
- Better for daily Docker usage

### Cons
- Members of the `docker` group effectively have high privileges on the system

---

## Important note

Being in the `docker` group is **similar to having root-level access in practice**.

That means:

- Only trusted users should be added to the `docker` group
- It is convenient, but not a small permission

---

## Which one should I use?

### Use `sudo` if:
- You are on a shared machine
- You want stricter control
- You use Docker only occasionally

### Use the `docker` group if:
- This is your personal machine or lab
- You use Docker frequently
- You want smoother daily workflow

---

## Recommended setup for learning

For a personal lab or learning environment:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

This is usually the most practical option.

---

## Common issue

If you see an error like:

```bash
permission denied while trying to connect to the Docker daemon socket
```

It usually means:

- You are not using `sudo`
- Your user is not in the `docker` group
- You added the group but have not refreshed your session yet

---

## Quick summary

- You do **not** need to become `root`
- `sudo docker ...` works
- Adding your user to the `docker` group is the common approach
- The `docker` group is convenient, but powerful