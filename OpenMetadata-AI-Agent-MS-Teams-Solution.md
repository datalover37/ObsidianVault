---
title: "AI Agent Solution for OpenMetadata Query & Search via MS Teams"
source: "[[Tavily Research]]"
author:
  - "[[Tim]]"
created: 2026-05-03
description: Complete solution architecture for building an AI agent that enables natural language querying of OpenMetadata through Microsoft Teams using MCP server integration
tags:
  - openmetadata
  - microsoft-teams
  - ai-agent
  - mcp-server
  - data-catalog
  - solution-architecture
  - enterprise-ai
---

# AI Agent Solution for OpenMetadata Query & Search via MS Teams

## Executive Summary

This document outlines a complete solution for building an AI-powered assistant that enables users to query, search, and summarize data catalog information from OpenMetadata through Microsoft Teams. The solution leverages the Model Context Protocol (MCP) to provide a standardized interface between AI agents and OpenMetadata's metadata repository.

**Key Benefits:**
- **Natural language queries** to OpenMetadata from Teams
- **Automated data discovery** and lineage exploration
- **Real-time metadata summaries** and reports
- **Seamless integration** with existing Microsoft 365 workflows
- **Enterprise-grade security** with OAuth-based authentication

---

## Solution Architecture

### High-Level Architecture

```
┌─────────────────┐
│  Microsoft      │
│  Teams User     │
└────────┬────────┘
         │ (1) Send message
         ▼
┌─────────────────────────────────┐
│   Microsoft Teams Bot           │
│   (Bot Framework SDK)           │
│   - Receives user messages      │
│   - OAuth authentication        │
│   - Message routing             │
└────────┬────────────────────────┘
         │ (2) Forward to AI Agent
         ▼
┌─────────────────────────────────┐
│   AI Agent Orchestrator         │
│   (LangChain / CrewAI)          │
│   - Natural language processing │
│   - Intent recognition          │
│   - Query planning              │
└────────┬────────────────────────┘
         │ (3) MCP Protocol
         ▼
┌─────────────────────────────────┐
│   OpenMetadata MCP Server       │
│   (mcp-server-openmetadata)     │
│   - Standardized API wrapper    │
│   - Tool definitions            │
│   - Resource management         │
└────────┬────────────────────────┘
         │ (4) REST API calls
         ▼
┌─────────────────────────────────┐
│   OpenMetadata Platform         │
│   - Metadata repository         │
│   - Search & discovery          │
│   - Lineage & quality data      │
└─────────────────────────────────┘
```

### Component Breakdown

#### 1. **Microsoft Teams Bot (Entry Point)**
- **Technology**: Microsoft Bot Framework SDK (Python/Node.js)
- **Responsibilities**:
  - Receive messages from Teams users
  - Handle OAuth authentication flow
  - Route messages to AI agent
  - Format and send responses back to Teams
  - Support @mentions, commands, and adaptive cards

#### 2. **AI Agent Orchestrator (Intelligence Layer)**
- **Technology**: LangChain, CrewAI, or AutoGen
- **Responsibilities**:
  - Parse natural language queries
  - Determine user intent (search, lineage, quality, glossary)
  - Plan multi-step workflows
  - Call appropriate MCP tools
  - Synthesize results into human-readable summaries
  - Handle context and conversation history

#### 3. **OpenMetadata MCP Server (Integration Layer)**
- **Technology**: `mcp-server-openmetadata` (Python)
- **Responsibilities**:
  - Expose OpenMetadata capabilities as MCP tools
  - Provide standardized interface for AI agents
  - Handle authentication with OpenMetadata
  - Manage API rate limiting and error handling
  - Support tools: search, lineage, data quality, glossary

#### 4. **OpenMetadata Platform (Data Layer)**
- **Technology**: OpenMetadata (open-source data catalog)
- **Responsibilities**:
  - Store and index metadata
  - Provide REST APIs for discovery
  - Maintain data lineage graph
  - Track data quality metrics
  - Manage business glossary

---

## Detailed Component Design

### 1. Microsoft Teams Bot Implementation

#### **Setup & Configuration**

**Prerequisites:**
- Azure subscription
- Microsoft Teams admin access
- Bot Framework registration

**Bot Registration:**
```bash
# Register bot in Azure Portal
az bot create \
  --resource-group openmetadata-rg \
  --name openmetadata-bot \
  --kind registration \
  --endpoint https://your-domain.com/api/messages
```

**Teams App Manifest:**
```json
{
  "manifestVersion": "1.16",
  "version": "1.0.0",
  "id": "your-app-id",
  "packageName": "com.yourcompany.openmetadata",
  "developer": {
    "name": "Your Company",
    "websiteUrl": "https://yourcompany.com",
    "privacyUrl": "https://yourcompany.com/privacy",
    "termsOfUseUrl": "https://yourcompany.com/terms"
  },
  "name": {
    "short": "OpenMetadata Assistant",
    "full": "OpenMetadata AI Assistant"
  },
  "description": {
    "short": "Query your data catalog with natural language",
    "full": "AI-powered assistant for searching and exploring OpenMetadata"
  },
  "bots": [
    {
      "botId": "your-bot-id",
      "scopes": ["personal", "team", "groupchat"],
      "supportsFiles": false,
      "isNotificationOnly": false,
      "commandLists": [
        {
          "scopes": ["personal", "team", "groupchat"],
          "commands": [
            {
              "title": "search",
              "description": "Search for tables, dashboards, or pipelines"
            },
            {
              "title": "lineage",
              "description": "Show data lineage for an entity"
            },
            {
              "title": "quality",
              "description": "Check data quality metrics"
            }
          ]
        }
      ]
    }
  ],
  "permissions": ["identity", "messageTeamMembers"],
  "validDomains": ["your-domain.com"]
}
```

#### **Bot Code (Python Example)**

```python
from botbuilder.core import BotFrameworkAdapter, TurnContext
from botbuilder.schema import Activity, ActivityTypes
from aiohttp import web
import asyncio

# Import AI agent orchestrator
from agent_orchestrator import OpenMetadataAgent

class OpenMetadataBot:
    def __init__(self, app_id: str, app_password: str):
        self.adapter = BotFrameworkAdapter(
            settings={"app_id": app_id, "app_password": app_password}
        )
        self.agent = OpenMetadataAgent()
    
    async def on_message_activity(self, turn_context: TurnContext):
        """Handle incoming messages from Teams"""
        user_message = turn_context.activity.text
        user_id = turn_context.activity.from_property.id
        
        # Show typing indicator
        await turn_context.send_activity(
            Activity(type=ActivityTypes.typing)
        )
        
        # Process with AI agent
        response = await self.agent.process_query(
            query=user_message,
            user_id=user_id,
            context=turn_context
        )
        
        # Send response back to Teams
        await turn_context.send_activity(response)
    
    async def on_turn(self, turn_context: TurnContext):
        """Main turn handler"""
        if turn_context.activity.type == ActivityTypes.message:
            await self.on_message_activity(turn_context)

# Web server setup
async def messages(req: web.Request):
    """Handle incoming HTTP requests from Teams"""
    body = await req.json()
    activity = Activity().deserialize(body)
    
    auth_header = req.headers.get("Authorization", "")
    
    async def call_bot(context: TurnContext):
        await bot.on_turn(context)
    
    await bot.adapter.process_activity(activity, auth_header, call_bot)
    return web.Response(status=200)

# Initialize bot
bot = OpenMetadataBot(
    app_id="YOUR_APP_ID",
    app_password="YOUR_APP_PASSWORD"
)

# Start web server
app = web.Application()
app.router.add_post("/api/messages", messages)
web.run_app(app, host="0.0.0.0", port=3978)
```

---

### 2. AI Agent Orchestrator Implementation

#### **Using LangChain + OpenMetadata MCP**

```python
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from langchain.tools import Tool
import httpx
import json

class OpenMetadataAgent:
    def __init__(self):
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0
        )
        self.mcp_url = "https://your-openmetadata.com/mcp"
        self.jwt_token = "YOUR_JWT_TOKEN"
        self.tools = self._initialize_mcp_tools()
        self.agent = self._create_agent()
    
    def _initialize_mcp_tools(self):
        """Initialize MCP tools from OpenMetadata"""
        # List available tools from MCP server
        response = httpx.post(
            self.mcp_url,
            json={
                "jsonrpc": "2.0",
                "id": 1,
                "method": "tools/list"
            },
            headers={
                "Authorization": f"Bearer {self.jwt_token}",
                "Content-Type": "application/json"
            }
        )
        
        mcp_tools = response.json()["result"]["tools"]
        
        # Convert MCP tools to LangChain tools
        langchain_tools = []
        for tool in mcp_tools:
            langchain_tools.append(
                Tool(
                    name=tool["name"],
                    description=tool["description"],
                    func=lambda query, tool_name=tool["name"]: 
                        self._call_mcp_tool(tool_name, query)
                )
            )
        
        return langchain_tools
    
    def _call_mcp_tool(self, tool_name: str, arguments: dict):
        """Call MCP tool and return result"""
        response = httpx.post(
            self.mcp_url,
            json={
                "jsonrpc": "2.0",
                "id": 2,
                "method": "tools/call",
                "params": {
                    "name": tool_name,
                    "arguments": arguments
                }
            },
            headers={
                "Authorization": f"Bearer {self.jwt_token}",
                "Content-Type": "application/json"
            }
        )
        
        result = response.json()["result"]
        return result["content"][0]["text"]
    
    def _create_agent(self):
        """Create LangChain agent with MCP tools"""
        prompt = ChatPromptTemplate.from_messages([
            ("system", """You are an AI assistant for OpenMetadata, 
            a data catalog platform. Help users discover, understand, 
            and govern their data assets.
            
            When users ask about:
            - Tables, databases, schemas → use search_metadata tool
            - Data lineage → use get_lineage tool
            - Data quality → use get_data_quality tool
            - Business terms → use search_glossary tool
            
            Always provide clear, concise answers with relevant links."""),
            MessagesPlaceholder(variable_name="chat_history"),
            ("human", "{input}"),
            MessagesPlaceholder(variable_name="agent_scratchpad")
        ])
        
        agent = create_openai_functions_agent(
            llm=self.llm,
            tools=self.tools,
            prompt=prompt
        )
        
        return AgentExecutor(
            agent=agent,
            tools=self.tools,
            verbose=True,
            max_iterations=5
        )
    
    async def process_query(self, query: str, user_id: str, context):
        """Process user query and return response"""
        try:
            result = await self.agent.ainvoke({
                "input": query,
                "chat_history": []  # Add conversation history here
            })
            
            return result["output"]
        
        except Exception as e:
            return f"Sorry, I encountered an error: {str(e)}"
```

---

### 3. OpenMetadata MCP Server Setup

#### **Installation**

```bash
# Install MCP server for OpenMetadata
pip install mcp-server-openmetadata

# Or use npx for Node.js version
npx -y mcp-server-openmetadata
```

#### **Configuration**

**Claude Desktop Config (`claude_desktop_config.json`):**
```json
{
  "mcpServers": {
    "openmetadata": {
      "command": "python",
      "args": ["-m", "mcp_server_openmetadata"],
      "env": {
        "OPENMETADATA_URL": "https://your-openmetadata.com",
        "OPENMETADATA_JWT_TOKEN": "your-jwt-token"
      }
    }
  }
}
```

#### **Available MCP Tools**

The OpenMetadata MCP server exposes the following tools:

1. **search_metadata**
   - Search for tables, databases, dashboards, pipelines
   - Parameters: `query`, `entity_type`, `limit`
   - Returns: List of matching entities with links

2. **get_lineage**
   - Retrieve data lineage for an entity
   - Parameters: `entity_fqn`, `depth`
   - Returns: Upstream and downstream lineage graph

3. **get_data_quality**
   - Get data quality test results
   - Parameters: `entity_fqn`, `test_suite`
   - Returns: Quality metrics and test status

4. **search_glossary**
   - Search business glossary terms
   - Parameters: `query`, `glossary_name`
   - Returns: Matching terms with definitions

5. **get_entity_details**
   - Get detailed information about an entity
   - Parameters: `entity_fqn`, `entity_type`
   - Returns: Full entity metadata

---

## Sample Use Cases & Queries

### Use Case 1: Data Discovery

**User Query (Teams):**
```
"Find all tables related to customer data"
```

**Agent Workflow:**
1. Parse intent: Search for tables
2. Call `search_metadata` tool with query="customer", entity_type="table"
3. Retrieve results from OpenMetadata
4. Format response with table names, descriptions, and links

**Response:**
```
I found 5 tables related to customer data:

1. **customers_raw** (Database: analytics_db)
   - Raw customer data from CRM
   - Owner: Data Engineering Team
   - [View in OpenMetadata](https://om.com/table/customers_raw)

2. **customers_enriched** (Database: analytics_db)
   - Enriched customer profiles with demographics
   - Owner: Analytics Team
   - [View in OpenMetadata](https://om.com/table/customers_enriched)

3. **customer_transactions** (Database: sales_db)
   - Customer purchase history
   - Owner: Sales Analytics
   - [View in OpenMetadata](https://om.com/table/customer_transactions)

Would you like to see the lineage for any of these tables?
```

---

### Use Case 2: Data Lineage Exploration

**User Query (Teams):**
```
"Show me the lineage for customers_enriched table"
```

**Agent Workflow:**
1. Parse intent: Get lineage
2. Call `get_lineage` tool with entity_fqn="analytics_db.customers_enriched"
3. Retrieve lineage graph
4. Format as visual representation

**Response:**
```
Here's the data lineage for **customers_enriched**:

**Upstream (Sources):**
├─ customers_raw (Table)
│  └─ CRM_API (Data Source)
├─ demographics_data (Table)
│  └─ External_Provider (Data Source)
└─ customer_segments (Table)

**Downstream (Consumers):**
├─ customer_360_view (Dashboard)
├─ churn_prediction_model (ML Model)
└─ marketing_campaigns (Pipeline)

**Transformation:**
- dbt model: `enrich_customers.sql`
- Last run: 2026-05-03 10:30 UTC
- Status: ✅ Success

[View full lineage in OpenMetadata](https://om.com/lineage/customers_enriched)
```

---

### Use Case 3: Data Quality Check

**User Query (Teams):**
```
"What's the data quality status for customers_enriched?"
```

**Agent Workflow:**
1. Parse intent: Check data quality
2. Call `get_data_quality` tool
3. Retrieve test results
4. Summarize pass/fail status

**Response:**
```
**Data Quality Report for customers_enriched**

**Overall Status:** ⚠️ Warning (2 tests failed)

**Test Results:**
✅ **Passed (8/10):**
- Column count validation
- Schema validation
- Null check on customer_id
- Unique constraint on email
- Data freshness (< 24h)
- Row count threshold
- Format validation on phone
- Range check on age

❌ **Failed (2/10):**
- **Null check on postal_code**: 3.2% null values (threshold: 1%)
- **Duplicate check on customer_id**: 15 duplicates found

**Last Run:** 2026-05-03 14:00 UTC
**Next Scheduled Run:** 2026-05-04 02:00 UTC

[View detailed report](https://om.com/quality/customers_enriched)

Would you like me to create a ticket for the data quality team?
```

---

### Use Case 4: Business Glossary Search

**User Query (Teams):**
```
"What does 'Customer Lifetime Value' mean in our glossary?"
```

**Agent Workflow:**
1. Parse intent: Search glossary
2. Call `search_glossary` tool with query="Customer Lifetime Value"
3. Retrieve term definition
4. Format with related terms

**Response:**
```
**Customer Lifetime Value (CLV)**

**Definition:**
The predicted net profit attributed to the entire future relationship with a customer. CLV helps determine how much to invest in acquiring and retaining customers.

**Formula:**
CLV = (Average Purchase Value × Purchase Frequency × Customer Lifespan) - Customer Acquisition Cost

**Related Terms:**
- Customer Acquisition Cost (CAC)
- Average Revenue Per User (ARPU)
- Churn Rate

**Used In:**
- customers_enriched.clv_score (Column)
- customer_analytics_dashboard (Dashboard)
- clv_prediction_model (ML Model)

**Owner:** Analytics Team
**Last Updated:** 2026-04-15

[View in Glossary](https://om.com/glossary/clv)
```

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up OpenMetadata instance
- [ ] Install and configure MCP server
- [ ] Test MCP tools with Claude Desktop
- [ ] Document available tools and capabilities

### Phase 2: Bot Development (Weeks 3-4)
- [ ] Register Microsoft Teams bot
- [ ] Implement basic message handling
- [ ] Set up OAuth authentication
- [ ] Deploy bot to Azure

### Phase 3: AI Agent Integration (Weeks 5-6)
- [ ] Implement LangChain agent orchestrator
- [ ] Connect agent to MCP server
- [ ] Add conversation memory
- [ ] Implement error handling

### Phase 4: Testing & Refinement (Weeks 7-8)
- [ ] User acceptance testing
- [ ] Performance optimization
- [ ] Add adaptive cards for rich responses
- [ ] Implement proactive notifications

### Phase 5: Production Deployment (Week 9)
- [ ] Security audit
- [ ] Load testing
- [ ] Documentation
- [ ] User training
- [ ] Go-live

---

## Technology Stack

### Required Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Data Catalog** | OpenMetadata | Metadata repository |
| **MCP Server** | mcp-server-openmetadata (Python) | Standardized API wrapper |
| **AI Framework** | LangChain / CrewAI | Agent orchestration |
| **LLM** | GPT-4 / Claude Opus | Natural language understanding |
| **Bot Framework** | Microsoft Bot Framework SDK | Teams integration |
| **Hosting** | Azure App Service / AWS Lambda | Bot hosting |
| **Authentication** | Azure AD OAuth 2.0 | User authentication |
| **Database** | PostgreSQL / MongoDB | Conversation history |

### Optional Enhancements

- **Vector Database** (Pinecone/Weaviate): Semantic search over metadata
- **Monitoring** (Application Insights): Bot performance tracking
- **Analytics** (Power BI): Usage analytics dashboard
- **Caching** (Redis): Response caching for common queries

---

## Security Considerations

### Authentication & Authorization

1. **OAuth 2.0 Flow**
   - Users authenticate via Azure AD
   - Bot receives delegated permissions
   - JWT tokens for OpenMetadata API

2. **Role-Based Access Control (RBAC)**
   - Respect OpenMetadata permissions
   - Users only see data they have access to
   - Audit all queries and actions

3. **Data Privacy**
   - No PII in conversation logs
   - Encrypt data in transit (TLS 1.3)
   - Encrypt data at rest (AES-256)

### Best Practices

- **Input Validation**: Sanitize all user inputs
- **Rate Limiting**: Prevent abuse (10 queries/min per user)
- **Error Handling**: Never expose internal errors to users
- **Audit Logging**: Log all queries with user ID and timestamp
- **Secret Management**: Use Azure Key Vault for credentials

---

## Cost Estimation

### Monthly Operating Costs (Estimated)

| Service | Usage | Cost |
|---------|-------|------|
| **Azure Bot Service** | Standard tier | $0.50/1000 messages |
| **Azure App Service** | B1 tier | $13/month |
| **OpenAI API** | GPT-4 (100K tokens/day) | $300/month |
| **OpenMetadata** | Self-hosted (VM) | $50/month |
| **Azure AD** | Free tier | $0 |
| **Storage** | 10GB | $2/month |
| **Total** | | **~$365/month** |

**For 50 users with ~20 queries/day each**

### Cost Optimization Tips

- Use GPT-3.5-turbo for simple queries (10x cheaper)
- Cache common responses (Redis)
- Implement query batching
- Use Azure Reserved Instances for compute

---

## Monitoring & Analytics

### Key Metrics to Track

1. **Usage Metrics**
   - Daily active users
   - Queries per user
   - Most common query types
   - Peak usage times

2. **Performance Metrics**
   - Average response time
   - MCP tool call latency
   - Bot availability (uptime)
   - Error rate

3. **Quality Metrics**
   - User satisfaction (thumbs up/down)
   - Query success rate
   - Escalation to human support
   - Conversation completion rate

### Monitoring Tools

- **Application Insights**: Bot telemetry
- **OpenMetadata Metrics**: API usage
- **Custom Dashboard**: Power BI for business metrics

---

## Sample Data & Testing

### Test Scenarios

#### Scenario 1: New User Onboarding
```
User: "Hi, I'm new to the data team. Can you help me find customer data?"
Bot: "Welcome! I can help you discover data in our catalog. Let me search for customer-related tables..."
[Shows top 5 tables with descriptions]
Bot: "Would you like to see the lineage for any of these tables?"
```

#### Scenario 2: Data Quality Investigation
```
User: "Why is the customer dashboard showing old data?"
Bot: "Let me check the data quality and freshness for the tables used in that dashboard..."
[Calls get_data_quality tool]
Bot: "I found that customers_enriched hasn't been updated in 36 hours. The scheduled pipeline failed. I've notified the data engineering team."
```

#### Scenario 3: Glossary Lookup
```
User: "What's the difference between MAU and DAU?"
Bot: "Let me look that up in our business glossary..."
[Searches glossary]
Bot: "MAU (Monthly Active Users) counts unique users over 30 days. DAU (Daily Active Users) counts unique users per day. MAU/DAU ratio indicates user engagement stickiness."
```

---

## Troubleshooting Guide

### Common Issues

#### Issue 1: Bot Not Responding
**Symptoms:** User sends message, no response
**Causes:**
- Bot service down
- Authentication failure
- Network timeout

**Solutions:**
1. Check Azure App Service status
2. Verify bot credentials in Azure Portal
3. Check Application Insights logs
4. Restart bot service

#### Issue 2: MCP Connection Failed
**Symptoms:** "Unable to connect to OpenMetadata"
**Causes:**
- Invalid JWT token
- OpenMetadata server down
- Network firewall blocking

**Solutions:**
1. Regenerate JWT token in OpenMetadata
2. Verify OpenMetadata URL is accessible
3. Check firewall rules
4. Test MCP endpoint with curl

#### Issue 3: Slow Response Times
**Symptoms:** Bot takes >10 seconds to respond
**Causes:**
- Large result sets from OpenMetadata
- LLM API latency
- No caching

**Solutions:**
1. Implement result pagination
2. Add Redis caching layer
3. Use faster LLM model for simple queries
4. Optimize MCP tool calls

---

## Future Enhancements

### Phase 2 Features (3-6 months)

1. **Proactive Notifications**
   - Alert users when data quality degrades
   - Notify when new datasets are added
   - Remind about stale documentation

2. **Advanced Analytics**
   - "Show me trending datasets this week"
   - "Which tables have the most quality issues?"
   - "What's the average data freshness across all pipelines?"

3. **Workflow Automation**
   - Create data quality tests via chat
   - Request access to datasets
   - Schedule data profiling jobs

4. **Multi-Modal Interactions**
   - Voice commands in Teams meetings
   - Visual lineage diagrams in chat
   - Interactive data previews

5. **Integration Expansion**
   - Slack integration
   - Email notifications
   - Mobile app support

---

## References & Resources

### Official Documentation

- **OpenMetadata Docs**: https://docs.open-metadata.org
- **OpenMetadata API**: https://docs.open-metadata.org/v1.12.x/api-reference
- **MCP Specification**: https://modelcontextprotocol.io
- **Microsoft Teams Bot**: https://learn.microsoft.com/en-us/microsoftteams/platform/bots
- **Bot Framework SDK**: https://github.com/microsoft/botframework-sdk

### GitHub Repositories

- **OpenMetadata**: https://github.com/open-metadata/OpenMetadata
- **MCP Server OpenMetadata**: https://github.com/yangkyeongmo/mcp-server-openmetadata
- **MCP Python SDK**: https://github.com/modelcontextprotocol/python-sdk
- **LangChain**: https://github.com/langchain-ai/langchain

### Community Resources

- **OpenMetadata Slack**: https://slack.open-metadata.org
- **MCP Discord**: https://discord.gg/mcp
- **Microsoft Teams Dev Community**: https://techcommunity.microsoft.com/teams

---

## Conclusion

This solution provides a comprehensive approach to building an AI-powered assistant for OpenMetadata that integrates seamlessly with Microsoft Teams. By leveraging the Model Context Protocol (MCP), the solution achieves:

✅ **Standardized Integration**: MCP provides a universal interface between AI agents and OpenMetadata  
✅ **Natural Language Access**: Users query metadata using plain English  
✅ **Enterprise Security**: OAuth-based authentication with RBAC  
✅ **Scalable Architecture**: Modular design supports future enhancements  
✅ **Cost-Effective**: ~$365/month for 50 users  

**Next Steps:**
1. Review architecture with stakeholders
2. Set up development environment
3. Begin Phase 1 implementation
4. Schedule weekly progress reviews

**Success Metrics (6 months post-launch):**
- 80% user adoption rate
- <5 second average response time
- 90% query success rate
- 4.5+ user satisfaction score

---

*Document Version: 1.0*  
*Last Updated: 2026-05-03*  
*Author: Tim*  
*Status: Ready for Implementation*
