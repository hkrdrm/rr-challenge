# Hacker Challenge - Multithreading Round-Robin
## reward $28 bonusly dollars

Here's our the worker that processes our bandwidth stuff. It runs on multiple worker boxes and each box runs 25 thread, so if 5 worker boxes
mutiplied by 25 threads gives us 125 threads.

```ruby
class BlastBandwidthWorker
  include Shoryuken::Worker

  shoryuken_options auto_delete: true

  def perform(sqs_message, payload)
    payload = JSON.parse(payload)
    blast   = BlastIndex[payload["blast_id"]]
    user    = blast.user

    BandwidthBlast
      .new(user)
      .send_message(blast_id: blast.id, to: payload["to"], message: blast.body)
  end
end
```

Here's our send_message method; we store the districts phone numbers (5 per district) in the districts table's meta field. Notice the .sample I'm
using here is a random approach to distributing the load across the phone numbers. If you run some test across a histogram you can see the
distribution is probably more random that the random generator .sample uses.

```ruby
  def send_message(blast_id:, to:, message:)
    district = BlastIndex[blast_id].district
    profile  = district.meta_hash[:bandwidth_profile]
    body     = MessageRequest.new

    body.application_id = application_id
    body.to             = [to]
    body.from           = profile[:numbers].sample
    body.text           = message

    res = messaging_client.create_message(account_id, body)

    BlastLog[blast_id: blast_id, to_number: to]
      .update(bandwidth_id: res.data.id)
  end
```

You are challenged to write a turbine utility that will distribute the elements of an array evenly across mutiple threads.
Your utility should take an array as an input and return an element of that array and maintain an even distribution across
many threads.

Hint: Utilize redis's ability to do atomic operations and implement a circular queue.
