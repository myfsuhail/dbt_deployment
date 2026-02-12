# Documentation Index

Welcome to the comprehensive documentation for the **dbt E-Commerce Analytics Testing & Deployment Showcase** project.

## Quick Navigation

### Getting Started
- **[README](../README.md)** - Project overview and quick start guide
- **[Setup Guide](SETUP.md)** - Detailed installation and configuration instructions

### Core Concepts
- **[Testing Guide](TESTING_GUIDE.md)** - Comprehensive testing strategies and best practices
- **[Deployment Guide](DEPLOYMENT_GUIDE.md)** - Orchestration patterns comparison (Airflow, Dagster, Prefect, CI/CD)
- **[Data Model Documentation](DATA_MODEL.md)** - Model-by-model documentation with lineage and business logic

### Technical Documentation
- **[Build Architecture](BUILD_GUIDE.md)** - Container architecture and Windows optimization details
- **[Project Structure](STRUCTURE.md)** - File organization and directory breakdown
- **[dbt Cloud Integration](DBT_CLOUD_GUIDE.md)** - Hybrid deployment with dbt Cloud API

### Presentations
- **[Testing Strategies Presentation](PRESENTATION_DBT_TESTS.md)** - Slide-style testing overview
- **[Deployment Patterns Presentation](PRESENTATION_DEPLOYMENT.md)** - Orchestration comparison slides

---

## Documentation by Use Case

### "I want to run this project locally"
1. Start with **[Setup Guide](SETUP.md)**
2. Follow the Docker orchestrator instructions
3. Reference **[Troubleshooting](SETUP.md#troubleshooting)** if issues arise

### "I want to understand the testing approach"
1. Read **[Testing Guide](TESTING_GUIDE.md)** for comprehensive coverage
2. Review **[Data Model](DATA_MODEL.md)** to see tests applied to each model
3. Check **[Testing Presentation](PRESENTATION_DBT_TESTS.md)** for quick overview

### "I want to choose an orchestration tool"
1. Read **[Deployment Guide](DEPLOYMENT_GUIDE.md)** for detailed comparison
2. Review the **[Comparison Matrix](DEPLOYMENT_GUIDE.md#comparison-matrix)**
3. See **[Deployment Presentation](PRESENTATION_DEPLOYMENT.md)** for executive summary

### "I want to understand the data models"
1. Start with **[Data Model Documentation](DATA_MODEL.md)**
2. Review the lineage graph and layer descriptions
3. Query the database using instructions in **[Setup Guide](SETUP.md#step-3-verify-data)**

### "I want to extend this project"
1. Understand the architecture in **[Data Model](DATA_MODEL.md)**
2. Follow testing patterns from **[Testing Guide](TESTING_GUIDE.md)**
3. Review **[Project Structure](STRUCTURE.md)** for file organization conventions

### "I want to deploy to production"
1. Review **[Deployment Guide](DEPLOYMENT_GUIDE.md#best-practices)**
2. Set up CI/CD following **[Strategy 4: GitHub Actions](DEPLOYMENT_GUIDE.md#strategy-4-cicd-with-github-actions)**
3. Consider **[dbt Cloud Integration](DBT_CLOUD_GUIDE.md)** for managed infrastructure

---

## Documentation Structure

```
docs/
├── README_DOCS.md              # This file - Documentation index
├── SETUP.md                    # Installation and local development
├── TESTING_GUIDE.md            # Testing strategies (schema + singular tests)
├── DEPLOYMENT_GUIDE.md         # Orchestration comparison (Airflow/Dagster/Prefect/CI-CD)
├── DATA_MODEL.md               # Model documentation with lineage
├── BUILD_GUIDE.md              # Docker architecture and technical details
├── STRUCTURE.md                # Project file organization
├── DBT_CLOUD_GUIDE.md          # dbt Cloud API integration
├── PRESENTATION_DBT_TESTS.md   # Testing presentation slides
└── PRESENTATION_DEPLOYMENT.md  # Deployment presentation slides
```

---

## Key Topics Covered

### Testing
- ✓ Schema tests (unique, not_null, accepted_values, relationships)
- ✓ Singular tests (custom SQL business logic)
- ✓ Test execution and debugging
- ✓ Store failures for investigation
- ✓ CI/CD integration
- ✓ Package-based tests (dbt_utils)

### Deployment
- ✓ Apache Airflow (task-based DAGs)
- ✓ Dagster (asset-centric workflows)
- ✓ Prefect (Python-native flows)
- ✓ GitHub Actions (CI/CD automation)
- ✓ Comparison matrix and selection guide
- ✓ Best practices for production

### Data Modeling
- ✓ Staging layer (data cleaning)
- ✓ Intermediate layer (business logic)
- ✓ Marts layer (analytics-ready tables)
- ✓ Dimensions and facts
- ✓ Data lineage
- ✓ Materialization strategies

### Technical Architecture
- ✓ Docker Compose multi-orchestrator setup
- ✓ Windows/Docker file-locking solutions
- ✓ Internal container path isolation
- ✓ Shared PostgreSQL database
- ✓ Sequential orchestrator usage

---

## Project Statistics

| Category | Count | Details |
|----------|-------|---------|
| **Documentation Files** | 10 | Comprehensive guides and presentations |
| **dbt Models** | 9 | Staging, intermediate, and marts layers |
| **dbt Seeds** | 3 | Raw CSV data files |
| **Schema Tests** | 25+ | YAML-defined generic tests |
| **Singular Tests** | 4 | Custom SQL business logic tests |
| **Orchestrators** | 3 | Airflow, Dagster, Prefect |
| **CI/CD** | 1 | GitHub Actions workflow |

---

## Learning Path

### Beginner
1. Read **[README](../README.md)** for project overview
2. Follow **[Setup Guide](SETUP.md)** to run locally
3. Explore **[Data Model](DATA_MODEL.md)** to understand the models
4. Review **[Testing Presentation](PRESENTATION_DBT_TESTS.md)** for quick testing overview

### Intermediate
1. Deep dive into **[Testing Guide](TESTING_GUIDE.md)**
2. Try all three orchestrators using **[Deployment Guide](DEPLOYMENT_GUIDE.md)**
3. Study **[Build Architecture](BUILD_GUIDE.md)** for technical details
4. Extend models and add custom tests

### Advanced
1. Implement CI/CD following **[GitHub Actions setup](DEPLOYMENT_GUIDE.md#strategy-4-cicd-with-github-actions)**
2. Integrate **[dbt Cloud](DBT_CLOUD_GUIDE.md)** for hybrid deployment
3. Adapt patterns for your production use case
4. Contribute improvements back to the project

---

## External Resources

### dbt Official Documentation
- [dbt Docs](https://docs.getdbt.com/) - Official documentation
- [dbt Discourse](https://discourse.getdbt.com/) - Community forum
- [dbt Slack](https://www.getdbt.com/community/) - Community chat

### Orchestration Tools
- [Apache Airflow Docs](https://airflow.apache.org/docs/)
- [Astronomer Cosmos](https://astronomer.github.io/astronomer-cosmos/) - Airflow + dbt integration
- [Dagster Docs](https://docs.dagster.io/)
- [dagster-dbt](https://docs.dagster.io/integrations/dbt) - Dagster + dbt integration
- [Prefect Docs](https://docs.prefect.io/)
- [prefect-dbt](https://prefecthq.github.io/prefect-dbt/) - Prefect + dbt integration

### Additional Learning
- [dbt Learn](https://learn.getdbt.com/) - Official courses
- [Analytics Engineering Guide](https://www.getdbt.com/analytics-engineering/) - Role overview
- [Modern Data Stack](https://www.getdbt.com/blog/future-of-the-modern-data-stack/) - Ecosystem overview

---

## Contributing

This is a demonstration project designed for learning and showcasing. To contribute:

1. **Documentation improvements** - Fix typos, add clarity, expand examples
2. **Code enhancements** - Add models, tests, or orchestrator integrations
3. **Bug reports** - Open issues on GitHub with detailed descriptions
4. **Feature requests** - Suggest new patterns or integrations to showcase

---

## Feedback & Support

- **Documentation Issues:** Open a GitHub issue with label `documentation`
- **Technical Issues:** Open a GitHub issue with label `bug`
- **Questions:** Check the troubleshooting sections or ask in dbt Discourse

---

## Version History

- **v1.0** - Initial release with Airflow, Dagster, Prefect, and GitHub Actions
- **v1.1** - Enhanced documentation with comprehensive testing and deployment guides
- **v1.2** - Added data model documentation and troubleshooting guides

---

**Start exploring:** Choose a link above based on your goal, or jump right into the **[Setup Guide](SETUP.md)** to get hands-on!
