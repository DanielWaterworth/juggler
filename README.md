Add jobs for asynchronous processing

    Juggler.throw(:method, params)

Add handlers, with optional concurrency, inside an EM loop

    EM.run {
      Juggler.juggle(:method, 10) do |deferrable, params|
        # Succeed the deferrable when the job is done
      end
    }

For example

    Juggler.juggle(:download, 10) do |df, params|
      http = EM::Protocols::HttpClient.request({
        :host => params[:host], 
        :port => 80, 
        :request => params[:path]
      })
      http.callback do |response|
        puts "Got response status #{response[:status]} for #{a}"
        df.success
      end
    end

The job is considered to have failed in the following cases:

* If the block raises an error
* If the block fails the passed deferrable
* If the job timeout is exceeded. In this case the passed deferrable will be failed by juggler. If you need to clean up any state in this case (for example you might want to cancel a HTTP request) then you should bind to `df.errback`.

## Customising behaviour

### Customising the back-off for failed jobs

By default, juggler will backoff jobs which failed exponentially using an exponent of 1.3, up to a maximum delay of 1 day, at which point the job will be buried. It's possible to customise this behaviour:

    Juggler.backoff_function = lambda { |job_runner, job_stats|
      # job_stats is a hash with string keys, as returned by beanstalkd's
      # stats-job command. Particularly useful stats in this context are:
      #
      # job_stats["age"] - the time since the put command that created this job
      # job_stats["delay"] - the previous amount of time delayed

      new_delay = ([1, job_stats["delay"] * 2].max).ceil
      if job_stats["age"] > 300
        job_runner.delete
        # Or you could bury the job
        # job_runner.bury
      else
        job_runner.release(new_delay)
      end
    }

## Important points to note

* If your deferrable code raises errors, this will not be handled by juggler.
