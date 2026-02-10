# 🛡️ Security Hardening — Implementation

This document outlines the security hardening measures applied across the
CyberLab v2 environment. The focus is on reducing attack surface, enforcing
least privilege, and improving visibility across the network.

Hardening steps were applied incrementally after core network functionality
was verified.

---

## pfSense Firewall

### Disable admin account

```pfsense
System -> User Manager
```

1. **Add** new user

2. Set username: ```jakef```

3. Set long password: ```zirzo5-qiZcik-megdis-wYgtor```

> These are just examples, I used other values for security reasons.

4. Enter your name

5. Add as a member of the **admins** group

6. ☑ Enable **Keep Command History**

7. Save

8. Edit the admin user settings

9. Check the box to disable admin account from logging in

10. Save

---

### Enable 2FA

next up
