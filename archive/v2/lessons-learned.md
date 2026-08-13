# Project Iteration (v2) - Lessons Learned

This lab was rebuilt to address early design limitations by improving IP addressing and network structure, strengthening documentation practices, and enhancing the overall security posture.

## Summary
This section outlines key lessons learned during the initial build of the lab and how they informed a cleaner redesign with improved network structure, documentation, and validation practices.

### Key Lessons Learned

- **Upfront network planning is critical**

  Initial IP addressing and network layout decisions were not planned and created unnecessary complexity. Rebuilding the lab reinforced the importance of defining IP schemes, VLANs, and trust boundaries before deployment.

- **Documentation should be created alongside implementation**

  Attempting to document the lab after major configuration changes led to gaps and inconsistencies. Writing documentation alongside applying configurations improved accuracy and reproducibility.

- **Network segmentation improves both security and troubleshooting**  
  
  Clear separation of management, user, and lab networks simplified firewall rule creation and made traffic flow easier to analyze and debug.

- **Rebuilding is sometimes faster than refactoring an existing setup**

  Refactoring an early, poorly planned design took more time than rebuilding the lab with a clean and well-planned architecture.

- **Verification steps help catch configuration issues early**

  Adding validation steps, such as connectivity tests, firewall rule checks, and interface verification, helped identify misconfigurations before they caused larger problems.
