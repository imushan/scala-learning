---
name: scala-hex-new
description: Use when user asks to create a new Scala project, scaffold a hexagonal architecture app, new hexagonal project, or mentions g8/giter8. Do NOT use for existing projects.
---

# Create new Scala hexagonal architecture project

Create a new Scala project using the Giter8 hexagonal architecture template.

## Usage
- User says: "create a new scala project" or "new hexagonal project" or "scaffold a scala app"

## Steps

1. Ask the user for project name (required) and other parameters:
   - `name` - Project name (required)
   - `organization` - Package organization (default: com.app)
   - `scala_version` - Scala version (default: 3.3.0)

2. Run the g8 command to create the project:
  <project-name> is a arguement
   ```bash
   g8 https://github.com/imushan/scala-bleep-jenkins-docker-template-v2.git  --name=<project-name> 
   ```

3. After creation, show the user:
   - Project location
   - How to compile: `cd <project-name> && bleep compile`
   - How to run: `bleep run app`
   - How to test: `bleep test tests`

## Template Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| name | Project name | myapp |
| description | Project description | A Scala hexagonal architecture application |
| organization | Organization/package | com.app |
| mainClass | Main class fully qualified name | $organization$.MainApp |
| scala_version | Scala version | 3.3.0 |
| acr_registry | ACR registry address | your-registry.cn-shanghai.cr.aliyuncs.com |
| acr_namespace | ACR namespace | your-namespace |
| git_repo | Git repository URL | https://gitee.com/your-user/your-repo.git |

### Start Metals MCP Server (after successful compile)
After a successful compile, start a Metals MCP server for the project if not already running:

```bash
# Check if MCP server is already running for this project
# Replace PROJECT_PATH with actual project path
pgrep -f "metals-mcp.*PROJECT_PATH" > /dev/null || \
  nohup metals-mcp --workspace PROJECT_PATH --port 11083 > /tmp/metals-mcp.log 2>&1 &
```

Example for a project at `/root/workspace/ai/langchain4s`:
```bash
pgrep -f "metals-mcp.*/root/workspace/ai/langchain4s" > /dev/null || \
  nohup metals-mcp --workspace /root/workspace/ai/langchain4s --port 11083 > /tmp/metals-mcp.log 2>&1 &
```


## Notes
- Prefer local template: `file:///opt/g8/scala-hexagonal-bleep-jenkins-docker.g8`
- Alternative GitHub: `https://github.com/imushan/scala-bleep-jenkins-docker-template-v2.git`
- Do NOT reinstall bleep or cs - they are pre-installed on the server