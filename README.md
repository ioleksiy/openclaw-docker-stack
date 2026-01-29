# docker-moltbot-cloudflare

Docker Compose project that deploys Moltbot behind a Cloudflare Tunnel for secure remote access without exposing ports on the host machine.

## Features

- **Moltbot**: AI-powered Telegram bot with gateway functionality
- **Cloudflare Tunnel**: Secure access without port forwarding or firewall configuration
- **Persistent Storage**: Moltbot config and workspace data persist across container restarts
- **Internal Network**: All services communicate through a private Docker network
- **No Exposed Ports**: No direct IP/port access from the host machine

## Prerequisites

1. A Cloudflare account with a domain
2. Cloudflare Zero Trust account (free tier available)
3. Docker Swarm initialized (Portainer handles this automatically)
4. Portainer installed
5. API keys for:
   - Anthropic (Claude)
   - OpenAI
   - Brave Search
   - Telegram Bot

## Setup Instructions

### 1. Create a Cloudflare Tunnel

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Access** > **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the connector
5. Give your tunnel a name (e.g., "moltbot-tunnel")
6. Copy the tunnel token that's generated
7. Configure a public hostname:
   - **Public hostname**: Your desired subdomain (e.g., `moltbot.yourdomain.com`)
   - **Service type**: HTTP
   - **URL**: `mbot:8000` (the internal Docker service name and port - adjust if Moltbot uses a different port)

### 2. Deploy with Portainer

1. In Portainer, go to **Stacks** > **Add stack**
2. Give your stack a **unique name** (e.g., `moltbot-production`, `moltbot-dev`, `moltbot-test`)
   - This stack name will automatically prefix all volumes and networks
3. Choose **Git Repository**
4. Enter your repository URL: `https://github.com/ioleksiy/docker-moltbot-cloudflare`
5. Set **Compose path** to: `docker-compose.yml`
6. In the **Environment variables** section, add:
   ```
   TUNNEL_TOKEN=your_actual_tunnel_token_here
   ANTHROPIC_API_KEY=your_anthropic_api_key_here
   OPENAI_API_KEY=your_openai_api_key_here
   BRAVE_API_KEY=your_brave_api_key_here
   TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
   GATEWAY_TOKEN=your_gateway_token_here
   ```
7. Click **Deploy the stack**

**Note**: Portainer deploys stacks using Docker Swarm mode, which provides automatic service discovery and load balancing.

### 3. Generate Secure Keys

For production deployments, generate strong random keys:

```bash
# Generate gateway token (32+ characters recommended)
openssl rand -hex 32

# Generate other secure tokens as needed
openssl rand -base64 32
```

### Multiple Stack Deployments

To deploy multiple instances from this repository:

1. Each Portainer stack **must** have a unique name
2. Each stack needs its own Cloudflare Tunnel token
3. Each stack should have unique API keys and gateway tokens
4. Examples:
   ```
   Stack Name: moltbot-production
   TUNNEL_TOKEN=token_for_prod
   ANTHROPIC_API_KEY=prod_anthropic_key
   TELEGRAM_BOT_TOKEN=prod_telegram_token
   GATEWAY_TOKEN=prod_gateway_token
   
   Stack Name: moltbot-development
   TUNNEL_TOKEN=token_for_dev
   ANTHROPIC_API_KEY=dev_anthropic_key
   TELEGRAM_BOT_TOKEN=dev_telegram_token
   GATEWAY_TOKEN=dev_gateway_token
   ```

Portainer automatically ensures each deployment has:
- Separate persistent volumes (prefixed with stack name)
- Isolated overlay networks
- Independent service instances
- Complete isolation between deployments

### 4. First Access to Moltbot

Once deployed, your Moltbot will be accessible through:
1. Your Telegram bot (using the configured bot token)
2. Your Cloudflare Tunnel URL (if web interface is available)

## Configuration

### Environment Variables

Set these in the Portainer stack interface:

**Required:**
- `TUNNEL_TOKEN`: Your Cloudflare Tunnel token
- `ANTHROPIC_API_KEY`: Your Anthropic (Claude) API key
- `OPENAI_API_KEY`: Your OpenAI API key
- `BRAVE_API_KEY`: Your Brave Search API key
- `TELEGRAM_BOT_TOKEN`: Your Telegram bot token
- `GATEWAY_TOKEN`: Gateway token for Moltbot

**Optional (with defaults):**
- `NODE_ENV`: Node environment (default: `production`)

### Persistent Storage

Data is stored in Docker volumes:
- `mbot_config`: Moltbot configuration and credentials
- `mbot_workspace`: Moltbot workspace files

This data persists even when containers are updated or reset.

### Network Security

Each stack creates its own isolated overlay network (automatically prefixed by Portainer) configured with:
- No ports exposed to the host
- Internal communication between services within the same stack
- Outbound internet access allowed (for Moltbot to connect to external services)
- Complete isolation between different stack deployments
- Swarm-scoped networking for service discovery

## Useful Commands

### View logs (Docker Swarm)
```bash
# List services
docker service ls

# View service logs
docker service logs <stack_name>_mbot
docker service logs <stack_name>_cloudflared
```

### Scale services
```bash
# Moltbot should stay at 1 replica for stateful operations
docker service scale <stack_name>_mbot=1
```

### Update services
```bash
# Update to latest images
docker service update --image ioleksiy/moltbot-docker:latest <stack_name>_mbot
docker service update --image cloudflare/cloudflared:latest <stack_name>_cloudflared
```

### Backup Moltbot data
```bash
# Replace 'moltbot-production' with your actual Portainer stack name
docker run --rm -v moltbot-production_mbot_config:/data -v $(pwd):/backup alpine tar czf /backup/mbot-config-backup.tar.gz -C /data .
docker run --rm -v moltbot-production_mbot_workspace:/data -v $(pwd):/backup alpine tar czf /backup/mbot-workspace-backup.tar.gz -C /data .
```

### Restore Moltbot data
```bash
# Replace 'moltbot-production' with your actual Portainer stack name
docker run --rm -v moltbot-production_mbot_config:/data -v $(pwd):/backup alpine tar xzf /backup/mbot-config-backup.tar.gz -C /data
docker run --rm -v moltbot-production_mbot_workspace:/data -v $(pwd):/backup alpine tar xzf /backup/mbot-workspace-backup.tar.gz -C /data
```

## Troubleshooting

### Service won't start
- Check logs: `docker service logs <stack_name>_mbot`
- Verify all required environment variables are set
- Ensure Cloudflare Tunnel is active in the dashboard
- Check service status: `docker service ps <stack_name>_mbot`

### Can't access Moltbot
- Verify the services are running: `docker service ls`
- Check Cloudflare Tunnel status in the Zero Trust dashboard
- Ensure public hostname is correctly configured to point to `mbot:8000` (or the appropriate port)
- Check service tasks: `docker service ps <stack_name>_cloudflared`

### Telegram bot not responding
- Verify the Telegram bot token is correct
- Check Moltbot logs: `docker service logs <stack_name>_mbot`
- Ensure all required API keys are set correctly
- Verify the bot has internet connectivity

### Data not persisting
- Check volumes exist: `docker volume ls | grep <your_portainer_stack_name>`
- Verify volumes are mounted: Check in Portainer UI under the stack's volumes tab
- Volumes in Docker Swarm persist across service updates

### Managing Multiple Deployments
- List all volumes: `docker volume ls`
- Each stack's volumes will be named: `<portainer_stack_name>_mbot_config` and `<portainer_stack_name>_mbot_workspace`
- To remove a specific deployment's data: 
  ```bash
  docker volume rm <portainer_stack_name>_mbot_config
  docker volume rm <portainer_stack_name>_mbot_workspace
  ```
- List all networks: `docker network ls`
- Each stack's network will be named: `<portainer_stack_name>_mbot_net`

## Security Notes

- HTTPS is automatically handled by Cloudflare
- No local ports are exposed
- Access is controlled through Cloudflare Zero Trust
- All services run on internal network only
- Use strong API keys and gateway tokens
- Consider adding Cloudflare Access policies for additional authentication
- Keep all images updated regularly
- **IMPORTANT**: Never commit API keys to version control!
- Store sensitive tokens securely in Portainer's environment variables

## Additional Resources

- [Moltbot Documentation](https://github.com/ioleksiy/moltbot-docker)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Portainer Documentation](https://docs.portainer.io/)
- [Telegram Bot API](https://core.telegram.org/bots/api)

## License

This project is licensed under the MIT License.

Free to use, modify, and fork by anyone!