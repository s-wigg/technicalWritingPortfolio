# cloud-billing-integration
CloudBillingIntegration - the Ruby gem for integrating with the Cloud Billing System

API for providing an integration point with Cloud Billing.

## Installation
### Build the gem

To build the Ruby code into a gem:

```shellgem 
build cloud-billing-integration.gemspec
```

### Install the gem locally or from Git
#### Install locally

```shellgem 
install ./cloud-billing-integration-0.1.0.gem
```

Add the following to the Gemfile:

    gem 'cloud-billing-integration', '~> 0.1.0'
    
#### Install from Git

Add the following to the Gemfile:

    gem 'cloud-billing-integration', :git => 'https://github.com/ccloud-billing-integration-ruby.git'
    
## Usage
This gem wraps the Cloud Integration service (a GoLang microservice) which is described in the [Documentation](./docs) and generated from the service's OpenAPI v2 spec.

### Creating a Billing Integration client

Create an instance of the client using the `client_id` and `client_secret` associated with the Okta application for the desired environment:

```ruby
# Load the gemrequire 'cloud-billing-integration'
# Setup Configuration and Authentication
configuration = CloudBillingIntegration::Configuration.new
configuration.server_index = CloudBillingIntegration::Configuration::Environment["integration"]
configuration.client_id = "OKTA_CLIENT_ID"
configuration.client_secret = "OKTA_CLIENT_SECRET"

# Setup API 
clientapi_client = CloudBillingIntegration::ApiClient.new(configuration)
# Setup Billing Integration client
client = CloudBillingIntegration::BillingIntegrationServiceApi.new(api_client)
```

The `server_index` field is the environment in which the client will be running (default is `production`). Your choices for the `server_index` field include:
- `CloudBillingIntegration::Configuration::Environment["local"]` allows you to test against a local instance of cloud-billing-integration-service.
- `CloudBillingIntegration::Configuration::Environment["development"]` allows you to test against Cloud accounts in the development environment without being billed.
- `CloudBillingIntegration::Configuration::Environment["integration"]` allows you to test against Cloud accounts in the integration environment without being billed.
- `CloudBillingIntegration::Configuration::Environment["production"]` is the production environment.

## Code Examples

### Register Payer Account
A Payer Account must be registered before any usage data can be sent to the billing system. This would typically be the first call when using the Gem.

```ruby
# Setup V1RegisterPayerAccountRequest 
requestregister_account_request = CloudBillingIntegration::V1RegisterPayerAccountRequest.new
register_account_request.organization_id = 'ORGANIZATION_ID'
register_account_request.contract_start = Time.now.utc.to_datetime.rfc3339 # contract_start format is RFC3339 (https://www.ietf.org/rfc/rfc3339.txt)

begin  
  # RegisterPayerAccount ensures the Payer Account is registered with the Cloud Billing system.  
  client.billing_integration_service_register_payer_account(register_account_request)
rescue CloudBillingIntegration::ApiError => e  
  puts "Exception when calling BillingIntegrationServiceApi->billing_integration_service_register_payer_account: #{e}"
end
```

### Meter Data
The billing system expects that ongoing metering or usage data be sent at regular time-based intervals once a Payer Account 
has been registered until billing is stopped for the Payer Account.

```ruby
require 'time'
require 'cloud-billing-integration'
meter_data_request = CloudBillingIntegration::V1MeterDataRequest.new # V1MeterDataRequest
meter_data_request.organization_id = 'ORGANIZATION_ID'
meter_data_request.datapoints = []

meter_data_request.datapoints << CloudBillingIntegration::V1Datapoint.new(  
  metric: CloudBillingIntegration::V1Metric::APPLIES,  
  value: 5,  
  timestamp: Time.now.utc.to_datetime.rfc3339
)
meter_data_request.datapoints << CloudBillingIntegration::V1Datapoint.new(  
  metric: CloudBillingIntegration::V1Metric::ADMIN_USERS,  
  value: 10,  timestamp: Time.now.utc.to_datetime.rfc3339
)
meter_data_request.datapoints << CloudBillingIntegration::V1Datapoint.new(  
  metric: CloudBillingIntegration::V1Metric::MAX_CONCURRENCY,  
  value: 7,  
  timestamp: Time.now.utc.to_datetime.rfc3339
)

begin  
  # MeterData sends the associated metering data to the Cloud Billing system.  
  client.billing_integration_service_meter_data(meter_data_request)
rescue CloudBillingIntegration::ApiError => e  
  puts "Error when calling BillingIntegrationServiceApi->billing_integration_service_meter_data: #{e}"
end
```

### Stop Billing
Stop billing is typically the final call in the billing lifecycle. 

```ruby
require 'time'
require 'cloud-billing-integration'

stop_billing_request = CloudBillingIntegration::V1StopBillingRequest.new # V1StopBillingRequest
stop_billing_request.organization_id = 'ORGANIZATION_ID'
stop_billing_request.timestamp = Time.now.utc.to_datetime.rfc3339 # timestamp format is RFC3339 (https://www.ietf.org/rfc/rfc3339.txt)

begin  
  # StopBilling ensures the Payer Account is no longer registered with the Cloud Billing system.  
  client.billing_integration_service_stop_billing(stop_billing_request)
rescue CloudBillingIntegration::ApiError => e  
  puts "Error when calling BillingIntegrationServiceApi->billing_integration_service_stop_billing: #{e}"
end
```

## Authentication
The authentication flow is a two step process. The gem will use the `client_id` and `client_secret` provided to perform an OAuth2 request to Okta to obtain a JWT
which is then sent in the request to the Billing Integration Service API where it is again authenticated/verified.

## Contributing

### Gem Changes
All of the code in [lib](./lib) is automatically generated by the [OpenAPI Generator](https://openapi-generator.tech) project using
the [custom template](./templates/ruby). If any changes are needed, they should go in one of the template files first and then run
`make openapi/generate` to regenerate the code in `lib`.

### API Changes
The swagger spec is generated using buf and can be recreated by running `make swagger/generate`.

### Publishing
Once the above changes are complete, the module needs to be repackaged for use:

1. Update the `gemVersion` in [openapi-config.yaml](./openapi-config.yaml).
2. Regenerate the ruby client via `make openapi/generate`.
3. Run `bundle` to bump the version in `Gemfile.lock`.
4. Create and merge a PR with the changes.
