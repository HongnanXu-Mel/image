### 3. System Design and Key Decisions

- **How the system is set up**：The app is split into two parts. The React front end talks to the Spring Boot back end through `/api` calls, and the back end saves data in a MySQL database with help from MyBatis. We run Spring Boot and MySQL in Docker during development so everyone works in the same environment and we can move to the cloud later without much drama.

- **What happens on the back end**：We keep the code simple by following a basic pattern: controllers receive the request, services handle the main work, and DAOs run the database queries. When someone creates a project, we wrap every insert (project, groups, students, rubric items) inside one transaction so the database stays clean even if something fails halfway.

- **Login and security decisions**：We use Spring Security with JWT tokens. Login and register are open, but everything else needs a valid token. Because we do not keep server sessions, the back end is ready for scaling out later, and classroom devices can swap users without carrying over old sessions.

- **What happens on the front end**：The front end runs on Umi + React + Ant Design. MobX stores keep user, subject, project, and comment data in sync across pages. A global wrapper restores login info from localStorage, shows a login modal if you are not signed in, and Axios adds the token to every call. If the token is no longer valid, we log the user out and refresh the page.

- **Data layout**：The database already has tables for users, students, subjects, templates, rubric items, comments, projects, groups, and group members. We ship sample records through `init.sql`, which makes demos easy and gives us material to build on when the scoring module is ready.

- **Why we picked these tools**：They match what the team already knows, so we deliver faster. REST APIs give us a clean bridge between front end and back end. JWT keeps requests stateless, which is handy for future load balancing. MobX is lightweight and covers our current workflow without a lot of setup.

- **What we still need to improve**：We want finer control of permissions, smoother handling of expired tokens (without a full reload), better performance for big imports, and environment-specific settings for the front end. We also plan to finish the scoring and report modules and plug them into LMS tools when the base features are stable.

