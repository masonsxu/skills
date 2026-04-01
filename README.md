# Masons Xu's Skills

Personal collection of Claude Agent Skills.

## Skills

### General

| Skill | Description |
|-------|-------------|
| [pretext-integration](./skills/pretext-integration) | Integrate @chenglou/pretext for fast, DOM-free multiline text measurement and layout |

### CloudWeGo Microservice

End-to-end development workflow for [CloudWeGo](https://www.cloudwego.io/) (Kitex + Hertz) microservice projects with clean architecture.

| Skill | Description |
|-------|-------------|
| [add-domain](./skills/cloudwego-microservice/add-domain) | Scaffold a new business domain module (Model→DAL→Converter→Logic→Wire) |
| [add-rpc-method](./skills/cloudwego-microservice/add-rpc-method) | End-to-end add a new RPC interface (IDL→all-layer implementation) |
| [add-http-endpoint](./skills/cloudwego-microservice/add-http-endpoint) | Add a new HTTP endpoint (IDL→gateway full flow) |
| [codegen](./skills/cloudwego-microservice/codegen) | Run code generation (Kitex/Hertz/Wire, with auto-detection) |
| [write-tests](./skills/cloudwego-microservice/write-tests) | Write tests following project conventions |
| [check-quality](./skills/cloudwego-microservice/check-quality) | Code quality checks (compile, lint, coverage, architecture compliance) |
| [diagnose](./skills/cloudwego-microservice/diagnose) | Classify errors, locate root causes, and provide fix plans |
| [gateway-logging](./skills/cloudwego-microservice/gateway-logging) | API gateway logging standards |

## Structure

```
skills/
  <skill-name>/SKILL.md                    # Standalone skill
  <group-name>/<skill-name>/SKILL.md       # Skill within a group
```

## Usage

Install a skill by copying its folder to your agent's skills directory (e.g., `~/.agents/skills/`).

For more information about the Agent Skills standard, see [agentskills.io](https://agentskills.io).
