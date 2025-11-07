### Describe Our Current System Architecture and Design Choices

Our setup keeps things simple: the React front end calls the Spring Boot back end through `/api`, and the back end stores data in MySQL with MyBatis. Spring Boot and MySQL both run in Docker during development so the team shares the same environment and can move to cloud hosting later with little change. On the back end we follow a straight-forward pattern—controllers handle the request, services run the logic, and DAOs talk to the database. Creating a project writes templates, groups, students, and rubric items inside one transaction so we do not end up with half-finished data. On the front end we use Umi + React + Ant Design, with MobX keeping user, subject, project, and comment data in sync. A global wrapper restores the login session from localStorage, shows a login modal when needed, and Axios adds the JWT token to each call.

### Justify the Critical Design Decisions

We picked Spring Boot, MyBatis, and MySQL because the team already knows them well, so we can build features faster and fix issues quickly. REST APIs give a clean bridge between the front end and back end and make it easy to add other clients later. JWT keeps requests stateless, which helps when we need to scale beyond one server or deal with shared classroom devices. On the front end, MobX is light to set up and matches our current flows better than heavier options, so we can keep the code base small while still sharing state across pages.

### Areas for Improvement and Possible Future Adjustments

We still need tighter role control and cleaner handling when tokens expire so the user does not lose their place. Big data imports should run faster, so we plan to look at batch inserts and better indexes. The front end should read its API base URL from the environment to support more than one deployment setup. Next term we also plan to finish the scoring and reporting modules, polish the PDF output, and look at connecting with the university LMS once the main flows feel stable.

