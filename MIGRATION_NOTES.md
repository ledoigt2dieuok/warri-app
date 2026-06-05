# Migration Notes: bli-app → warri-app

**Migration Date**: June 5, 2026  
**Status**: Complete  
**Archived Repository**: [bli-app](https://github.com/ledoigt2dieuok/bli-app)

---

## Overview

The Wari App project has been successfully migrated from `bli-app` to `warri-app`. This document explains the transition and key information for developers.

---

## What Changed

### Repository Status

| Aspect | Old (bli-app) | New (warri-app) |
|--------|---------------|-----------------|
| Status | **🔴 Archived** | **🟢 Active** |
| URL | github.com/ledoigt2dieuok/bli-app | github.com/ledoigt2dieuok/warri-app |
| Default Branch | main | main |
| Issues | Closed to new | Accepting |
| PRs | Closed | Accepting |

### Why the Migration?

1. **Naming**: "warri" better reflects the app purpose (West African money transfers)
2. **Fresh Start**: Clean slate for production-ready development
3. **Consolidation**: Single source of truth for the codebase
4. **Branding**: Aligns with product naming

---

## For Developers

### Updating Your Local Setup

```bash
# Remove old remote
git remote remove origin

# Add new remote
git remote add origin https://github.com/ledoigt2dieuok/warri-app.git

# Update branch tracking
git fetch origin
git branch -u origin/main main

# Verify
git remote -v
```

### Updating Documentation Links

If you have bookmarks or documentation pointing to bli-app:

**Old**: `https://github.com/ledoigt2dieuok/bli-app`  
**New**: `https://github.com/ledoigt2dieuok/warri-app`

### Environment & Configuration

All `.env` files, credentials, and configurations remain the same:
- Database credentials ✅
- API keys ✅
- Service configurations ✅
- Docker Compose setups ✅

### Branch Structure

```
warri-app/
├── main               # Production-ready code
├── develop            # Development branch
├── feature/*          # Feature branches
├── bugfix/*           # Bug fix branches
└── hotfix/*           # Hotfix branches
```

### Commit History

✅ **All commits from bli-app are preserved**

Total commits migrated: **30+**

Notable commits:
- Core implementation framework
- Payment provider integrations
- Testing setup
- CI/CD pipelines
- Documentation

### Pull Requests & Issues

- **Old PRs (bli-app)**: No longer accessible, review warri-app for status
- **New PRs (warri-app)**: All future work tracked here
- **Issues**: New issues should be created in warri-app

---

## Project Structure (Unchanged)

The project structure remains identical:

```
warri-app/
├── src/                 # Frontend (React Native)
├── backend/             # Backend (Node.js/Express)
├── tests/               # Test files
├── docs/                # Documentation
├── .github/workflows/   # CI/CD
├── docker-compose.yml   # Docker setup
├── package.json         # Dependencies
└── README.md           # Project overview
```

---

## Key Features (All Preserved)

✅ User Authentication  
✅ Money Transfer  
✅ Payment Providers (Orange, MTN, Wave)  
✅ Transaction History  
✅ Analytics Dashboard  
✅ Redux State Management  
✅ Jest Testing  
✅ CI/CD Pipelines  

---

## Development Workflow

### Creating a Feature Branch

```bash
# Update main branch
git checkout main
git pull origin main

# Create feature branch
git checkout -b feature/my-feature

# Make changes, commit, push
git push origin feature/my-feature

# Create PR on GitHub
```

### Commit Message Convention

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Example**:
```
feat(biometric): Add fingerprint authentication

Added biometric authentication support for iOS and Android using expo-local-authentication.
Includes fallback to PIN-based authentication.

Closes #123
```

### PR Process

1. Create feature branch from `develop`
2. Make changes and commit
3. Push to GitHub
4. Create Pull Request
5. Await code review
6. Make requested changes if any
7. Merge to `develop`
8. Merge `develop` → `main` for release

---

## CI/CD Pipeline

### Automated on Every PR

✅ Linting (ESLint)  
✅ Type checking (TypeScript)  
✅ Unit tests (Jest)  
✅ Security scanning  
✅ Coverage reports  

### Automated on Merge to Main

✅ Full test suite  
✅ Build verification  
✅ Staging deployment (auto)  
✅ Production deployment (manual approval)  

---

## Team Communication

### Important Links

- **Repository**: https://github.com/ledoigt2dieuok/warri-app
- **Issues**: https://github.com/ledoigt2dieuok/warri-app/issues
- **PRs**: https://github.com/ledoigt2dieuok/warri-app/pulls
- **Discussions**: https://github.com/ledoigt2dieuok/warri-app/discussions

### Discussion Topics

- Feature requests → Issues with `feature` label
- Bug reports → Issues with `bug` label
- Questions → Discussions tab
- Architecture → Discussions tab

---

## Frequently Asked Questions

### Q: Can I still access bli-app?

**A**: The repository is archived and read-only. You can view the history but cannot make changes. All development continues in warri-app.

### Q: Do I need to re-clone the repository?

**A**: No, update your existing clone:
```bash
git remote set-url origin https://github.com/ledoigt2dieuok/warri-app.git
git fetch origin
```

### Q: Are all features from bli-app in warri-app?

**A**: ✅ Yes, 100% of the code has been migrated with full commit history.

### Q: What about existing pull requests in bli-app?

**A**: They are archived. If they contained work in progress, the relevant changes are in warri-app main branch.

### Q: Do API endpoints change?

**A**: No, the backend API remains the same. Hosting and deployment may change, but endpoints are compatible.

### Q: What about databases and migrations?

**A**: All database migrations and schemas are unchanged. Refer to the same migration files in warri-app.

### Q: Can I contribute?

**A**: Absolutely! Follow the [CONTRIBUTING.md](CONTRIBUTING.md) guide in warri-app.

---

## Migration Checklist

### For Development Team

- [ ] Updated local git remotes
- [ ] Pulled latest from warri-app
- [ ] Reviewed new branch structure
- [ ] Updated IDE bookmarks
- [ ] Read DEVELOPMENT_PLAN.md
- [ ] Verified API compatibility
- [ ] Tested local setup

### For DevOps/Deployment Team

- [ ] Updated CI/CD pipeline URLs
- [ ] Configured staging environment
- [ ] Configured production environment
- [ ] Updated monitoring/logging
- [ ] Updated backup procedures
- [ ] Tested deployment pipeline

### For Documentation Team

- [ ] Updated all external links
- [ ] Updated API documentation
- [ ] Updated setup guides
- [ ] Updated contributing guide
- [ ] Updated troubleshooting docs

### For QA/Testing Team

- [ ] Reviewed test suite
- [ ] Updated test documentation
- [ ] Configured test environments
- [ ] Verified CI/CD test runs
- [ ] Created test plan for new phases

---

## What's Next

See [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md) for:

- **Phase 1**: New Features (Biometric Auth, Offline Mode, etc.)
- **Phase 2**: Code Improvements
- **Phase 3**: Enhanced Documentation
- **Phase 4**: Test Coverage Expansion
- **Phase 5**: CI/CD & Deployment
- **Phase 6**: Production Launch

---

## Support & Questions

- **Documentation**: Check [docs/](docs/) folder
- **Issues**: Create issue in warri-app
- **Discussions**: Use GitHub Discussions
- **Email**: support@wari-app.local (for urgent matters)

---

## Important Reminders

⚠️ **Do NOT** commit to archived bli-app repository  
✅ **DO** create issues/PRs in warri-app  
✅ **DO** follow the development plan  
✅ **DO** keep documentation updated  

---

**Migration completed successfully!**

All systems are go for warri-app development. 🚀

**Last Updated**: June 5, 2026  
**Next Review**: Ongoing as new phases commence
