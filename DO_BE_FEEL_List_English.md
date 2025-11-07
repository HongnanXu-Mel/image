### Describe Our Current System Architecture and Design Choices

- **Front end**：Built with Umi + React + Ant Design. MobX stores manage user, subject, project, and comment data. A top-level wrapper loads login info from localStorage, shows a modal when login is needed, and Axios adds JWT tokens to every request.
- **Back end**：Spring Boot exposes REST endpoints under `/api`, supported by a layered design where controllers receive calls, services do the work, and MyBatis DAOs hit a MySQL database. When creating a project, the back end writes template, group, student, and rubric records in one transaction. Spring Boot and MySQL run in Docker for consistent local environments.

### Justify the Critical Design Decisions

- **Front end**：We chose React + Ant Design because the team can ship views quickly and keep them consistent. MobX is light, so sharing state across pages stays simple without extra boilerplate.
- **Back end**：Spring Boot, MyBatis, and MySQL match our experience, letting us deliver features fast. REST APIs keep the contract clear for every client, and JWT makes each call stateless, which helps with shared classroom devices and future scaling.

### Areas for Improvement and Possible Future Adjustments

- **Front end**：Improve token-expiry handling so users do not lose their current screen, read the API base URL from environment variables, and finish the scoring/report UI flows.
- **Back end**：Add finer-grained role checks, batch imports for large CSV files, better indexing, and hook the reporting logic into LMS integrations once the core flows are stable.

