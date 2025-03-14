# Building a Multi-Agent System with OpenAI Agents SDK

This guide provides a complete example of building a customer support system with multiple specialized agents using the OpenAI Agents SDK.

## Overview

The multi-agent customer support system includes:

1. **Triage Agent**: The initial agent that categorizes customer issues and directs them to specialized agents
2. **Billing Agent**: Specializes in handling billing and payment inquiries
3. **Technical Agent**: Handles technical troubleshooting
4. **General Support Agent**: Handles general inquiries and account management

The system also implements:
- Tools for accessing customer information and performing actions
- Guardrails to ensure appropriate responses
- Handoffs between agents
- Tracing for monitoring the system

## Implementation

### Step 1: Define Customer Data Models

First, let's define our data models:

```python
import asyncio
from typing import Optional, List, Dict, Any
from pydantic import BaseModel, Field
from datetime import datetime

from agents import tool, RunContextWrapper

# ----- Data Models -----

class CustomerInfo(BaseModel):
    customer_id: str
    name: str
    email: str
    account_type: str
    signup_date: str
    subscription_tier: Optional[str] = None
    payment_status: Optional[str] = None

class BillingInfo(BaseModel):
    customer_id: str
    plan: str
    monthly_cost: float
    next_billing_date: str
    payment_method: str
    past_due: bool = False
    
class TechnicalIssue(BaseModel):
    issue_id: str
    title: str
    description: str
    severity: str
    status: str
    created_date: str
```

### Step 2: Set Up Mock Data

```python
# ----- Mock Database -----

CUSTOMERS = {
    "C1001": CustomerInfo(
        customer_id="C1001",
        name="Alice Johnson",
        email="alice@example.com",
        account_type="Business",
        signup_date="2023-05-15",
        subscription_tier="Premium",
        payment_status="Active"
    ),
    "C1002": CustomerInfo(
        customer_id="C1002",
        name="Bob Smith",
        email="bob@example.com",
        account_type="Personal",
        signup_date="2024-01-10",
        subscription_tier="Basic",
        payment_status="Past Due"
    )
}

BILLING_INFO = {
    "C1001": BillingInfo(
        customer_id="C1001",
        plan="Premium",
        monthly_cost=49.99,
        next_billing_date="2025-04-15",
        payment_method="Credit Card"
    ),
    "C1002": BillingInfo(
        customer_id="C1002",
        plan="Basic",
        monthly_cost=9.99,
        next_billing_date="2025-04-10",
        payment_method="PayPal",
        past_due=True
    )
}

TECHNICAL_ISSUES = {
    "T2001": TechnicalIssue(
        issue_id="T2001",
        title="Login Error",
        description="Unable to log in despite correct credentials",
        severity="Medium",
        status="Open",
        created_date="2025-03-10"
    ),
    "T2002": TechnicalIssue(
        issue_id="T2002",
        title="Dashboard Not Loading",
        description="Dashboard shows spinning icon but never loads",
        severity="High",
        status="In Progress",
        created_date="2025-03-12"
    )
}
```

### Step 3: Define Tools

```python
# ----- Tools -----

@tool
async def lookup_customer(customer_id: str) -> str:
    """
    Look up a customer's basic information.
    
    Args:
        customer_id: The customer's ID
        
    Returns:
        The customer's information or an error message if not found
    """
    await asyncio.sleep(0.5)  # Simulate database lookup
    
    if customer_id in CUSTOMERS:
        customer = CUSTOMERS[customer_id]
        return f"Customer found:\n" + \
               f"Name: {customer.name}\n" + \
               f"Email: {customer.email}\n" + \
               f"Account Type: {customer.account_type}\n" + \
               f"Signup Date: {customer.signup_date}"
    else:
        return f"Customer ID {customer_id} not found."

@tool
async def get_billing_info(customer_id: str) -> str:
    """
    Get a customer's billing information.
    
    Args:
        customer_id: The customer's ID
        
    Returns:
        The customer's billing information or an error message if not found
    """
    await asyncio.sleep(0.5)  # Simulate database lookup
    
    if customer_id in BILLING_INFO:
        billing = BILLING_INFO[customer_id]
        status = "Past Due" if billing.past_due else "Current"
        
        return f"Billing Information for {CUSTOMERS[customer_id].name}:\n" + \
               f"Plan: {billing.plan}\n" + \
               f"Monthly Cost: ${billing.monthly_cost:.2f}\n" + \
               f"Next Billing Date: {billing.next_billing_date}\n" + \
               f"Payment Method: {billing.payment_method}\n" + \
               f"Status: {status}"
    else:
        return f"Billing information for customer ID {customer_id} not found."

@tool
async def get_technical_issues(customer_id: str) -> str:
    """
    Get a list of technical issues for a customer.
    
    Args:
        customer_id: The customer's ID
        
    Returns:
        A list of the customer's technical issues or a message if none exist
    """
    await asyncio.sleep(0.5)  # Simulate database lookup
    
    # Mock logic to associate customers with issues
    customer_issues = {}
    customer_issues["C1001"] = ["T2001"]
    customer_issues["C1002"] = ["T2002"]
    
    if customer_id in customer_issues:
        issue_ids = customer_issues[customer_id]
        if not issue_ids:
            return f"No technical issues found for customer {customer_id}."
        
        issues_text = "Technical Issues:\n"
        for issue_id in issue_ids:
            issue = TECHNICAL_ISSUES[issue_id]
            issues_text += f"- {issue.title} (ID: {issue.issue_id})\n"
            issues_text += f"  Severity: {issue.severity}, Status: {issue.status}\n"
            issues_text += f"  Description: {issue.description}\n"
        
        return issues_text
    else:
        return f"No technical issues found for customer {customer_id}."

@tool
async def process_payment(context: RunContextWrapper, customer_id: str, amount: float) -> str:
    """
    Process a payment for a customer.
    
    Args:
        customer_id: The customer's ID
        amount: The payment amount
        
    Returns:
        Confirmation of the payment processing
    """
    await asyncio.sleep(1)  # Simulate payment processing
    
    if customer_id in CUSTOMERS:
        # In a real system, this would connect to a payment processor
        today = datetime.now().strftime("%Y-%m-%d")
        transaction_id = f"TX-{customer_id}-{today}"
        
        # Update mock database
        if customer_id in BILLING_INFO and BILLING_INFO[customer_id].past_due:
            BILLING_INFO[customer_id].past_due = False
        
        return f"Payment of ${amount:.2f} processed successfully for {CUSTOMERS[customer_id].name}.\n" + \
               f"Transaction ID: {transaction_id}\n" + \
               f"Date: {today}\n" + \
               f"Account status updated to: Current"
    else:
        return f"Payment processing failed. Customer ID {customer_id} not found."

@tool
async def update_technical_issue(issue_id: str, status: str) -> str:
    """
    Update the status of a technical issue.
    
    Args:
        issue_id: The ID of the technical issue
        status: The new status to set (Open, In Progress, Resolved, Closed)
        
    Returns:
        Confirmation of the update
    """
    await asyncio.sleep(0.5)  # Simulate database update
    
    valid_statuses = ["Open", "In Progress", "Resolved", "Closed"]
    if status not in valid_statuses:
        return f"Invalid status. Must be one of: {', '.join(valid_statuses)}"
    
    if issue_id in TECHNICAL_ISSUES:
        old_status = TECHNICAL_ISSUES[issue_id].status
        TECHNICAL_ISSUES[issue_id].status = status
        
        return f"Issue {issue_id} updated.\n" + \
               f"Title: {TECHNICAL_ISSUES[issue_id].title}\n" + \
               f"Status changed from '{old_status}' to '{status}'."
    else:
        return f"Issue ID {issue_id} not found."
```

### Step 4: Define Guardrails

```python
from agents import input_guardrail, output_guardrail, GuardrailFunctionOutput

@input_guardrail
def validate_customer_inquiry(context, agent, input):
    """Validate that customer inquiries contain appropriate content."""
    
    # Convert input to string if it's a list
    input_text = input if isinstance(input, str) else str(input)
    
    # Check for inappropriate language
    inappropriate_terms = ["stupid", "idiot", "hate", "terrible"]
    for term in inappropriate_terms:
        if term.lower() in input_text.lower():
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info="Input contains inappropriate language"
            )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

@output_guardrail
def ensure_professional_tone(context, agent, output):
    """Ensure agent responses maintain a professional tone."""
    
    output_text = str(output)
    
    # Check for unprofessional phrases
    unprofessional_phrases = ["not my problem", "can't help you", "figure it out"]
    for phrase in unprofessional_phrases:
        if phrase.lower() in output_text.lower():
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info="Response contains unprofessional language"
            )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

@output_guardrail
def verify_customer_data_handling(context, agent, output):
    """Verify that customer data is handled appropriately."""
    
    output_text = str(output)
    
    # Check if output contains both customer ID and other sensitive info
    contains_customer_id = any(customer_id in output_text for customer_id in CUSTOMERS.keys())
    contains_email = any(customer.email in output_text for customer in CUSTOMERS.values())
    
    if contains_customer_id and contains_email:
        # Output includes both customer ID and email, which might be too much exposed info
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info="Response contains sensitive customer identification information"
        )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

### Step 5: Create Specialized Agents

```python
from agents import agent, handoff

# Billing Agent
billing_agent = agent(
    """You are a billing specialist at our customer support center.
    You help customers with subscription plans, payments, invoices, refunds, and billing-related questions.
    Always be courteous and professional. If a customer has a past due payment, encourage them to update their payment information.
    If you need to look up customer information, use the lookup_customer and get_billing_info tools.
    If a customer wants to make a payment, use the process_payment tool.
    
    For technical issues or account management questions, suggest transferring to the appropriate specialist.
    """,
    tools=[lookup_customer, get_billing_info, process_payment],
    input_guardrails=[validate_customer_inquiry],
    output_guardrails=[ensure_professional_tone, verify_customer_data_handling]
)

# Technical Support Agent
technical_agent = agent(
    """You are a technical support specialist at our customer support center.
    You help customers troubleshoot issues with our product, walk them through solutions, and record technical issues.
    Always be patient and thorough in your explanations. Use clear, step-by-step instructions.
    If you need to look up customer information, use the lookup_customer tool.
    If you need to check or update technical issues, use the get_technical_issues and update_technical_issue tools.
    
    For billing questions or account management issues, suggest transferring to the appropriate specialist.
    """,
    tools=[lookup_customer, get_technical_issues, update_technical_issue],
    input_guardrails=[validate_customer_inquiry],
    output_guardrails=[ensure_professional_tone, verify_customer_data_handling]
)

# General Support Agent
general_support_agent = agent(
    """You are a general support specialist at our customer support center.
    You help customers with account management, general information about our products and services, and other inquiries.
    Always be helpful and informative. If you don't know the answer to a question, be honest about it.
    If you need to look up customer information, use the lookup_customer tool.
    
    For specific billing or technical questions, suggest transferring to the appropriate specialist.
    """,
    tools=[lookup_customer],
    input_guardrails=[validate_customer_inquiry],
    output_guardrails=[ensure_professional_tone, verify_customer_data_handling]
)
```

### Step 6: Create Handoffs and Triage Agent

```python
# Create handoffs
billing_handoff = handoff(
    billing_agent,
    tool_name_override="transfer_to_billing",
    tool_description_override="Transfer to a billing specialist for questions about payments, subscriptions, or invoices."
)

technical_handoff = handoff(
    technical_agent,
    tool_name_override="transfer_to_technical",
    tool_description_override="Transfer to a technical specialist for help with product issues or troubleshooting."
)

general_handoff = handoff(
    general_support_agent,
    tool_name_override="transfer_to_general_support",
    tool_description_override="Transfer to a general support specialist for account management or general information."
)

# Add all handoffs to each agent
billing_agent_with_handoffs = billing_agent.with_handoffs([technical_handoff, general_handoff])
technical_agent_with_handoffs = technical_agent.with_handoffs([billing_handoff, general_handoff])
general_agent_with_handoffs = general_support_agent.with_handoffs([billing_handoff, technical_handoff])

# Triage Agent
triage_agent = agent(
    """You are a triage specialist at our customer support center.
    Your job is to understand the customer's issue and direct them to the appropriate specialist:
    
    - For questions about billing, payments, subscriptions, or invoices, transfer to a billing specialist.
    - For technical problems, product troubleshooting, or bug reports, transfer to a technical specialist.
    - For account management questions or general information, transfer to a general support specialist.
    
    If you're unsure which specialist is appropriate, ask clarifying questions before making a transfer.
    Always be polite and professional. Begin by asking how you can help the customer today.
    """,
    tools=[],  # Triage agent doesn't need tools directly
    input_guardrails=[validate_customer_inquiry],
    output_guardrails=[ensure_professional_tone]
)

# Add handoffs to triage agent
triage_agent_with_handoffs = triage_agent.with_handoffs([
    billing_handoff, 
    technical_handoff, 
    general_handoff
])
```

### Step 7: Set Up Tracing and Run the System

```python
from agents.tracing import add_trace_processor
from agents.tracing.processors import ConsoleSpanExporter, BatchTraceProcessor
from agents import Runner
from agents.exceptions import InputGuardrailTripwireTriggered, OutputGuardrailTripwireTriggered

# Set up console exporter for tracing
console_exporter = ConsoleSpanExporter()
console_processor = BatchTraceProcessor(console_exporter)
add_trace_processor(console_processor)

async def handle_customer_inquiry(customer_input: str, customer_context: dict = None):
    """Handle a customer inquiry using our multi-agent system."""
    
    # Create context with customer information if provided
    context = customer_context or {}
    
    # Create runner
    runner = Runner()
    
    try:
        # Start with the triage agent
        result = await runner.run_async(
            triage_agent_with_handoffs, 
            customer_input,
            context=context
        )
        
        # Return the final result
        return {
            "success": True,
            "response": result.output,
            "agent_path": [item.agent.name for item in result.run_items if hasattr(item, 'agent')]
        }
        
    except InputGuardrailTripwireTriggered as e:
        return {
            "success": False,
            "error": "Input guardrail triggered",
            "details": e.guardrail_result.output.output_info
        }
        
    except OutputGuardrailTripwireTriggered as e:
        return {
            "success": False,
            "error": "Output guardrail triggered",
            "details": e.guardrail_result.output.output_info
        }
        
    except Exception as e:
        return {
            "success": False,
            "error": "Unexpected error",
            "details": str(e)
        }
```

### Step 8: Example Usage

```python
import asyncio

async def main():
    # Example 1: Billing inquiry
    billing_result = await handle_customer_inquiry(
        "I need help with my subscription payment. My customer ID is C1002.",
        {"user_id": "U12345"}
    )
    print("Billing Inquiry Result:")
    print(billing_result)
    print("\n" + "-"*50 + "\n")
    
    # Example 2: Technical issue
    technical_result = await handle_customer_inquiry(
        "I'm having trouble logging into my account. It keeps saying 'invalid credentials'.",
        {"user_id": "U12346"}
    )
    print("Technical Inquiry Result:")
    print(technical_result)
    print("\n" + "-"*50 + "\n")
    
    # Example 3: General inquiry
    general_result = await handle_customer_inquiry(
        "I'd like to know more about your premium plan features.",
        {"user_id": "U12347"}
    )
    print("General Inquiry Result:")
    print(general_result)
    print("\n" + "-"*50 + "\n")
    
    # Example 4: Inappropriate content (guardrail should trigger)
    inappropriate_result = await handle_customer_inquiry(
        "This is a stupid product and I hate your company!",
        {"user_id": "U12348"}
    )
    print("Inappropriate Content Result:")
    print(inappropriate_result)

if __name__ == "__main__":
    asyncio.run(main())
```

## Expected System Flow

1. User sends an inquiry to the system
2. The triage agent categorizes the inquiry and determines the appropriate specialist
3. Control is handed off to the specialist agent via the handoff mechanism
4. The specialist agent uses tools to access relevant information and respond to the user
5. If needed, the specialist can handoff to another agent for further assistance
6. Guardrails ensure all interactions remain appropriate and professional
7. Tracing provides visibility into the entire process for monitoring and debugging

## Benefits of This Multi-Agent Approach

1. **Specialization**: Each agent can focus on a specific domain, improving response quality
2. **Seamless Handoffs**: Users experience a coherent conversation despite interacting with multiple agents
3. **Efficient Triage**: The triage agent quickly directs users to the right specialist
4. **Safety and Compliance**: Guardrails ensure all interactions meet company standards
5. **Observability**: Tracing provides insights into the system's behavior
6. **Flexibility**: New agents and tools can be added to expand capabilities

## Further Enhancements

This example could be extended with:

1. Integration with actual databases and APIs
2. Additional specialized agents (e.g., sales, onboarding)
3. More sophisticated guardrails for domain-specific requirements
4. User authentication and authorization
5. Conversational memory for multi-turn interactions
6. Analytics and reporting on agent performance
7. A/B testing different agent prompts and tools
