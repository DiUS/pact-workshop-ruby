Example Ruby Project for Pact Workshop
======================================

When writing a lot of small services, testing the interactions between these becomes a major headache. That's the problem Pact is trying to solve.

Integration tests typically are slow and brittle, requiring each component to have it's own environment to run the tests in. With a micro-service architecture, this becomes even more of a problem. They also have to be 'all-knowing' and this makes them difficult to keep from being fragile.

After J. B. Rainsberger's talk "Integrated Tests Are A Scam" people have been thinking how to get the confidence we need to deploy our software to production without having a tiresome integration test suite that does not give us all the coverage we think it does.

Pact is a ruby gem that allows you to define a pact between service consumers and providers. It provides a DSL for service consumers to define the request they will make to a service producer and the response they expect back. This expectation is used in the consumers specs to provide a mock producer, and is also played back in the producer specs to ensure the producer actually does provide the response the consumer expects.

This allows you to test both sides of an integration point using fast unit tests.

#Example Pact use#

Given we have a client that needs to make a HTTP GET request to a sinatra webapp, and requires a response in JSON format. The client would look something like:

client.rb:

    require 'httparty'
    require 'uri'
    require 'json'

    class Client


      def load_provider_json
        response = HTTParty.get(URI::encode('http://localhost:8081/producer.json?valid_date=' + Time.now.httpdate))
        if response.success?
          JSON.parse(response.body)
        end
      end


    end

and the provider:
provider.rb

    require 'sinatra/base'
    require 'json'


    class Provider < Sinatra::Base


      get '/provider.json', :provides => 'json' do
        valid_time = Time.parse(params[:valid_date])
        JSON.pretty_generate({
          :test => 'NO',
          :valid_date => DateTime.now,
          :count => 1000
        })
      end

    end


This provider expects a valid_date parameter in HTTP date format, and then returns some simple json back.

Running the client with the following rake task against the provider works nicely:

    desc 'Run the client'
    task :run_client => :init do
      require 'client'
      require 'ap'
      ap Client.new.load_provider_json
    end


    $ rake run_client
    http://localhost:8081/provider.json?valid_date=Thu,%2015%20Aug%202013%2003:15:15%20GMT
    {
              "test" => "NO",
        "valid_date" => "2013-08-15T13:31:39+10:00",
             "count" => 1000
    }


Now lets get the client to use the data it gets back from the provider. Here is the updated client method that uses the returned data:

client.rb

      def process_data
        data = load_provider_json
        ap data
        value = data['count'] / 100
        date = Time.parse(data['date'])
        puts value
        puts date
        [value, date]
      end

Add a spec to test this client:

client_spec.rb:

    require 'spec_helper'
    require 'client'


    describe Client do


      let(:json_data) do
        {
          "test" => "NO",
          "date" => "2013-08-16T15:31:20+10:00",
          "count" => 100
        }
      end
      let(:response) { double('Response', :success? => true, :body => json_data.to_json) }


      it 'can process the json payload from the provider' do
        HTTParty.stub(:get).and_return(response)
        expect(subject.process_data).to eql([1, Time.parse(json_data['date'])])
      end


    end

Let's run this spec and see it all pass:

    $ rake spec
    /Users/ronald/.rvm/rubies/ruby-1.9.3-p448/bin/ruby -S rspec ./spec/client_spec.rb


    Client
    http://localhost:8081/producer.json?valid_date=Fri,%2016%20Aug%202013%2005:44:41%20GMT
    {
         "test" => "NO",
         "date" => "2013-08-16T15:31:20+10:00",
        "count" => 100
    }
    1
    2013-08-16 15:31:20 +1000
      can process the json payload from the provider


    Finished in 0.00409 seconds
    1 example, 0 failures

However, there is a problem with this integration point. The provider returns a 'valid_date' while the consumer is trying to use 'date', which will blow up when run for real even with the tests all passing. Here is where Pact comes in.

#Pact to the rescue#

Lets setup Pact in the consumer. Pact lets the consumers define the expectations for the integration point.

spec_helper.rb:

    Pact.service_consumer "My Consumer" do
      has_pact_with "My Provider" do
        mock_service :my_provider do
          port 1234
        end
      end
    end

This defines a consumer and a producer that runs on port 1234.

The spec for the client now has a pact section.

client_spec.rb:

    describe 'pact with provider', :pact => true do

      let(:date) { Time.now.httpdate }

      before do
        my_provider.
          given("provider is in a sane state").
            upon_receiving("a request for provider json").
              with({
                  method: :get,
                  path: '/provider.json',
                  query: URI::encode('valid_date=' + date)
              }).
              will_respond_with({
                status: 200,
                headers: { 'Content-Type' => 'application/json' },
                body: json_data
              })
      end

      it 'can process the json payload from the provider' do
        expect(subject.process_data).to eql([1, Time.parse(json_data['date'])])
      end

    end


Running this spec still passes, but it creates a pact file which we can use to validate our assumptions on the provider side.

    $ rake spec
    /Users/ronald/.rvm/rubies/ruby-1.9.3-p448/bin/ruby -S rspec ./spec/client_spec.rb


    Client
    http://localhost:8081/provider.json?valid_date=Fri,%2016%20Aug%202013%2006:09:44%20GMT
    {
      "test"  => "NO",
      "date"  => "2013-08-16T15:31:20+10:00",
      "count" => 100
    }
    1
    2013-08-16 15:31:20 +1000
      can process the json payload from the provider
      pact with provider
    http://localhost:8081/provider.json?valid_date=Fri,%2016%20Aug%202013%2006:09:44%20GMT
    {
      "test"  => "NO",
      "date"  => "2013-08-16T15:31:20+10:00",
      "count" => 100
    }
    1
    2013-08-16 15:31:20 +1000
        can process the json payload from the provider


Generated pact file (spec/pacts/my_consumer-my_provider.json):

    {
      "provider": {
        "name": "My Producer"
      },
      "consumer": {
        "name": "My Consumer"
      },
      "interactions": [
        {
          "description": "a request for provider json",
          "request": {
            "method": "get",
            "path": "/provider.json",
            "query": "valid_date=Fri,%2016%20Aug%202013%2006:09:44%20GMT"
          },
          "response": {
            "status": 200,
            "headers": {
              "Content-Type": "application/json"
            },
            "body": {
              "test": "NO",
              "date": "2013-08-16T15:31:20+10:00",
              "count": 100
            }
          },
          "provider_state": "provider is in a sane state"
        }
      ],
      "metadata": {
        "date": "2013-08-16T16:09:44+10:00",
        "pact_gem": {
          "version": "0.1.23"
        }
      }
    }

#Provider Setup#

Pact has a rake task to verify the producer against the generated pact file. It can get the pact file from any URL (like the last successful CI build), but we just going to use the local one. Here is the addition to the Rakefile.

Rakefile:

    require 'pact/tasks'

spec/service_consumers/pact_helper.rb:

    require 'pact/provider/rspec'

    Pact.service_provider "My Provider" do

      honours_pact_with 'My Consumer' do

        # This example points to a local file, however, on a real project with a continuous
        # integration box, you would use a [Pact Broker](https://github.com/bethesque/pact_broker) or publish your pacts as artifacts,
        # and point the pact_uri to the pact published by the last successful build.

        pact_uri '../pacts/my_consumer-my_provider.json'
      end
    end

Now if we run our pact verification task, it should fail.

    $ rake pact:verify


    Pact in spec/pacts/my_consumer-my_provider.json
      Given producer is in a sane state
        a request for provider json to /provider.json
          returns a response which
            has status code 200
            has a matching body (FAILED - 1)
            includes headers
              "Content-Type" with value "application/json" (FAILED - 2)


    Failures:


      1) Pact in spec/pacts/my_consumer-my_provider.json Given provider is in a sane state a request for provider json to /provider.json returns a response which has a matching body
         Failure/Error: expect(parse_entity_from_response(last_response)).to match_term response['body']
           {
             "date"  => {
               :expected => "2013-08-16T15:31:20+10:00",
               :actual   => nil
             },
             "count" => {
               :expected => 100,
               :actual   => 1000
             }
           }

Looks like we need to update the producer to return 'date' instead of 'valid_date', we also need to update the client expectation to return 1000 for the count and the correct content type (we expected application/json but got application/json;charset=utf-8). Doing this, and we now have fast unit tests on each side of the integration point instead of tedious integration tests.
