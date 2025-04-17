---
status: ai-generated
last updated:
  "{ DATE }": 
version: "0.1"
tags:
  - workflow
  - gherkin
  - bdd
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.

# Gherkin Feature Template

This template provides a standard structure for creating Gherkin feature files. When using this template, copy the content into a `.feature` file in the `/features` directory.

## Basic Structure

```gherkin
Feature: {{FEATURE_NAME}}
  As a {{USER_ROLE}}
  I want to {{ACTION}}
  So that {{BENEFIT}}

  Background:
    Given {{COMMON_PRECONDITION_1}}
    And {{COMMON_PRECONDITION_2}}

  Scenario: {{SCENARIO_NAME_1}}
    Given {{PRECONDITION}}
    When {{ACTION}}
    Then {{EXPECTED_OUTCOME}}
    And {{ADDITIONAL_OUTCOME}}

  Scenario: {{SCENARIO_NAME_2}}
    Given {{PRECONDITION}}
    When {{ACTION}}
    Then {{EXPECTED_OUTCOME}}
```

## Using Scenario Outlines for Data-Driven Testing

```gherkin
  Scenario Outline: {{SCENARIO_OUTLINE_NAME}}
    Given {{PRECONDITION_WITH_<PLACEHOLDER>}}
    When {{ACTION_WITH_<PLACEHOLDER>}}
    Then {{OUTCOME_WITH_<PLACEHOLDER>}}

    Examples:
      | {{PLACEHOLDER_1}} | {{PLACEHOLDER_2}} |
      | {{VALUE_1_1}}     | {{VALUE_1_2}}     |
      | {{VALUE_2_1}}     | {{VALUE_2_2}}     |
      | {{VALUE_3_1}}     | {{VALUE_3_2}}     |
```

## Using Tags

```gherkin
@{{CATEGORY_TAG}}
Feature: {{FEATURE_NAME}}

  @{{TAG_1}} @{{TAG_2}}
  Scenario: {{SCENARIO_NAME}}
    Given {{PRECONDITION}}
    When {{ACTION}}
    Then {{EXPECTED_OUTCOME}}
```

## Full Example

```gherkin
@payment-system
Feature: Process Credit Card Payments
  As a customer
  I want to pay for my order with a credit card
  So that I can complete my purchase quickly

  Background:
    Given I have items in my shopping cart
    And I am on the checkout page

  @happy-path @credit-card
  Scenario: Successful payment with valid credit card
    Given I have entered valid credit card information
    When I click the \"Pay Now\" button
    Then I should see a payment confirmation message
    And I should receive an email receipt

  @error-handling @credit-card
  Scenario: Payment declined due to insufficient funds
    Given I have entered credit card information with insufficient funds
    When I click the \"Pay Now\" button
    Then I should see an \"Insufficient funds\" error message
    And I should have the option to try another payment method

  @data-validation @credit-card
  Scenario Outline: Validate credit card number format
    Given I am on the payment form
    When I enter \"<card_number>\" in the card number field
    Then I should see the validation message \"<message>\"

    Examples:
      | card_number       | message                         |
      | 4111111111111111  | Card number validated           |
      | 411111111111111   | Card number must be 16 digits   |
      | ABCDEFGHIJKLMNOP  | Card number must contain digits |
```

## Best Practices

1. **Feature Title**: Should clearly describe the feature being tested.

2. **User Story Format**: Include the \"As a... I want to... So that...\" structure to provide context.

3. **Background**: Use for common setup steps that apply to all scenarios in the feature.

4. **Scenarios**: 
   - Name scenarios clearly and descriptively
   - Each scenario should test one specific behavior
   - Keep scenarios independent of each other

5. **Steps**:
   - Start with Given (precondition), When (action), Then (outcome)
   - Use And for additional conditions or outcomes
   - Write steps in user language, not technical implementation

6. **Tags**:
   - Use tags to categorize scenarios
   - Common tags include @smoke, @regression, @wip (work in progress)
   - Use custom tags for feature areas or specific types of tests

7. **Scenario Outlines**:
   - Use for data-driven tests
   - Placeholders should be surrounded by < and >
   - Provide meaningful examples in the Examples table

## Common Gherkin Steps for Our Project

### Authentication Steps
```gherkin
Given I am logged in as a \"<role>\"
Given I am not logged in
When I log in with valid credentials
When I log in with invalid credentials
Then I should be redirected to the dashboard
Then I should see an error message
```

### Navigation Steps
```gherkin
Given I am on the \"<page_name>\" page
When I navigate to the \"<page_name>\" page
When I click on the \"<element_name>\"
Then I should be redirected to the \"<page_name>\" page
```

### Form Interaction Steps
```gherkin
Given I have filled in the \"<form_name>\" form
When I enter \"<value>\" in the \"<field_name>\" field
When I select \"<option>\" from the \"<dropdown_name>\" dropdown
When I check the \"<checkbox_name>\" checkbox
When I submit the form
Then the form should be submitted successfully
Then I should see a validation error
```

### Data Verification Steps
```gherkin
Then I should see \"<text>\" on the page
Then the \"<element>\" should contain \"<value>\"
Then the \"<element>\" should be visible
Then the \"<element>\" should not be visible
```

## Document History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 0.1 | {{DATE}} | Initial draft | {{AUTHOR}} |
| {{VERSION}} | {{DATE}} | {{CHANGES}} | {{AUTHOR}} |