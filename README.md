# Calculator REST API

A Java 21, Spring Boot, and Maven calculator service for the SPE mini-project, with a browser frontend and a Jenkins/Docker Compose delivery pipeline.

## Project report and local evidence

- [Mini-project report (PDF)](docs/report.pdf)
- [Editable report source](docs/report.html)
- [Calculator UI screenshot](docs/evidence/calculator-ui.png)
- [Local health, API, and Compose verification record](docs/evidence/local-verification.txt)

The local evidence records the running Compose stack and sample API response. Jenkins build/email screenshots and the GitHub push-to-deploy demo must be captured from the configured external services after a successful pipeline run.

## Build and test

```powershell
mvn clean test
mvn package
```

JUnit XML reports are written to `target/surefire-reports/`.

## Git and GitHub contribution workflow

Keep `main` deployable and make each change on a short-lived branch. For example:

```bash
git switch main
git pull --ff-only
git switch -c feature/describe-your-change
# make and review your changes
git diff
git add <files>
git commit -m "Describe the change"
git push -u origin feature/describe-your-change
```

Open a pull request from the branch to `main`, review the diff and CI results, then merge it on GitHub. Use focused commits with clear messages so `git log --oneline --graph --decorate` tells the project's story; use `git show <commit>` to inspect a change and `git revert <commit>` to undo a shared commit safely.

Never commit passwords, access tokens, private keys, or populated `.env` files. Store deployment credentials in Jenkins credentials, use `.env.example` for safe variable names and placeholders, and check `git status` before committing. If a secret was committed, removing it in a later commit is not enough: revoke or rotate it immediately.

## Run

```powershell
mvn spring-boot:run
```

The API listens on `http://localhost:8081` when run directly.
Open `http://localhost:8081` in a browser to use the calculator frontend.

## Run with Docker Hub image

The published application image is:

```text
certifiedunc/spe-calculator:latest
```

Run the calculator image directly and map it to port 8081:

```powershell
docker login
docker pull certifiedunc/spe-calculator:latest
docker run --rm -d --name spe-calculator -p 8081:8081 certifiedunc/spe-calculator:latest
```

Verify it:

```powershell
curl http://localhost:8081/actuator/health
curl "http://localhost:8081/api/v1/add?a=2&b=3"
docker stop spe-calculator
```

## Run with Docker Compose

Docker Compose starts the calculator and Nginx reverse proxy and waits for their health checks. To build and run the local source:

```powershell
docker compose up --build -d
docker compose ps
```

To use the Docker Hub image instead of building locally, set `CALCULATOR_IMAGE`:

```powershell
$env:CALCULATOR_IMAGE="certifiedunc/spe-calculator:latest"
docker compose pull calculator
docker compose up -d --no-build
docker compose ps
```

Open `http://localhost:8082` for the frontend and API. The calculator listens on port 8081 only inside the Compose network. To stop the stack, run `docker compose down`.

## Publish the image to Docker Hub

Use a Docker Hub access token rather than an account password. Set the Docker Hub namespace once; this repository uses `certifiedunc`.

```powershell
docker login -u certifiedunc
docker build -t certifiedunc/spe-calculator:latest .
docker push certifiedunc/spe-calculator:latest
```

Add a build-specific immutable tag before pushing to production:

```powershell
$tag = (Get-Date -Format 'yyyyMMdd-HHmmss')
docker build -t "certifiedunc/spe-calculator:$tag" .
docker push "certifiedunc/spe-calculator:$tag"
```

The tagged and `latest` versions should appear at `https://hub.docker.com/r/certifiedunc/spe-calculator`.

## API

All calculator endpoints accept decimal query parameters `a` and `b` and return a JSON object containing the operation, operands, and result.

```text
GET http://localhost:8081/api/v1/add?a=2&b=3
GET http://localhost:8081/api/v1/subtract?a=7&b=3
GET http://localhost:8081/api/v1/multiply?a=2.5&b=4
GET http://localhost:8081/api/v1/divide?a=7&b=2
GET http://localhost:8081/actuator/health
```

Missing or non-numeric parameters and division by zero return HTTP 400 with a JSON error response. Division results use `MathContext.DECIMAL128` precision.

## Run Jenkins in Docker

Jenkins itself is containerized, so setup does not depend on a locally installed Jenkins service.

```bash
export DOCKER_GID="$(stat -c '%g' /var/run/docker.sock)"
docker compose -f docker-compose.jenkins.yml up -d --build
```

Open `http://localhost:8090`, complete the initial administrator setup, and create a Multibranch Pipeline job for this repository. The Jenkins container mounts the Docker Engine socket and has Maven, Docker CLI, and Docker Compose v2 available. The socket gives Jenkins control of the host Docker daemon, so keep Jenkins access restricted to trusted administrators.

The Jenkins image installs the Git, Pipeline, GitHub Branch Source, Workspace Cleanup, JUnit, and Email Extension plugins at build time. Configure the Jenkins credentials described below.

## Jenkins pipeline

The declarative `Jenkinsfile` checks out, validates Compose, builds, tests, archives the executable JAR, and builds a Docker image for every branch and pull request. Only builds for the `main` branch push the versioned image and `latest` tag to Docker Hub and deploy the stack with Docker Compose. Deployment waits for the calculator's `/actuator/health` check and the proxy health check to pass.

### Jenkins setup

1. The Jenkins image installs the Git, Pipeline, GitHub Branch Source, Workspace Cleanup, JUnit, and Email Extension plugins when it is built.
2. The container already provides Java 21, Maven, Docker CLI, and Docker Compose; no local agent tools or host Jenkins installation are required.
3. Add a Jenkins **Username with password** credential with ID `dockerhub-credentials`. Use your Docker Hub username and a Docker Hub access token as the password.
4. Create a **Multibranch Pipeline** job for this GitHub repository. Configure it to discover branches and pull requests, and ensure it loads the `Jenkinsfile` from each branch.
5. Configure a GitHub webhook for push and pull request events and enable branch and pull request discovery in the multibranch source. The Jenkinsfile also polls each branch every five minutes as a fallback.
6. Ensure the Jenkins agent can run Docker and Docker Compose v2, and can bind host port 8082. The deployment runs on that agent's Docker host. Jenkins is available on host port 8090 (container port 8080).
7. Configure Jenkins SMTP so it can send mail:
   - In **Manage Jenkins → System → Extended E-mail Notification**, set SMTP server `smtp.gmail.com`, SMTP port `465`, enable SSL, and add your Gmail username and app password.
   - In the Jenkins Mailer section, enter the same Gmail address and configure `smtp.gmail.com` with port `465` and SSL.
   - Create a Gmail **App Password** from Google's account security page. Do not use your normal Gmail password or store the password in Git.
   - The pipeline sends success, failure, unstable, and aborted notifications to `akshat0akt52@gmail.com` by default. Set the Jenkins environment variable `NOTIFICATION_EMAIL` to override that address. Notification failures are recorded without masking the build result.
8. In GitHub, protect `main` by requiring pull requests before merge. Once Jenkins status checks are publishing to GitHub, require the Jenkins check before allowing a merge. Keep deployment credentials in Jenkins, never in the repository.

The multibranch pipeline verifies feature branches and pull requests but does not merge them. Review and merge the pull request on GitHub; the resulting `main` build is the only build that pushes and deploys. The deployed app is available at `http://localhost:8082`.

## Capture the Jenkins success screenshot

After starting a build:

1. Open the build page in Jenkins at **Dashboard → SPE_Mini_Project → main → Build History → green build number**.
2. Select **Stage View** to show the green pipeline stages, or open **Console Output** for the full successful log.
3. Press `Windows + Shift + S`, drag over the Jenkins window, and save the PNG through the Snipping Tool.
4. Include the build number and each stage in the screenshot. A successful pipeline run may show all stages in green or blue depending on the Jenkins theme.
5. Keep a second screenshot of the received email from `akshat0akt52@gmail.com`'s inbox, using the Jenkins build subject and number as evidence.
