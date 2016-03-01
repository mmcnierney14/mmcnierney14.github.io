---
layout: post
title:  "Testing Reading and Writing Files in RSpec"
date:   2016-02-29
---

Let's say you need to write a Ruby program that takes in a CSV of number pairs and outputs another CSV with those number pairs summed. Easy. You whip this up in 5 minutes:

{% highlight ruby %}
#!/usr/bin/env ruby
require 'csv'

in_file_path = ARGV[0]
out_file_path = ARGV[1]

# Sum each pair of numbers
sums = CSV.readlines(in_file_path).map do |num1, num2|
  num1.to_i + num2.to_i
end

# Write the result to a file
CSV.open(out_file_path, "wb") do |csv|
  sums.each { |sum| csv << [sum] }
end
{% endhighlight %}

This works really well. Running `./add_numbers_untestable.rb numbers.csv sums.csv` takes in

{% highlight text %}
10,10
15,10
20,10
11,11
22,22
{% endhighlight %}

and outputs

{% highlight text %}
20
25
30
22
44
{% endhighlight %}

You commit the code and push it to master, satisfied with a job well done. As it turns out, the script is extremely helpful to your coworkers. They begin to use it on a daily basis, and as it becomes more popular, you realize that you should probably write specs for it to make sure it doesn't break in the future.

You quickly realize that this code is untestable. How can you call a script from RSpec? How can you pass in a fake input file and output file? How can you make sure the numbers are being summed properly?

## Making It Testable

To write a spec for the script, we first need to make some modifications to the script to make it easy to test. RSpec is good at testing classes, so let's put the business logic of the script into a class:

{% highlight ruby %}
#!/usr/bin/env ruby
require 'csv'

class AddNumbers
  attr_accessor :in_file_path, :out_file_path

  def initialize(args)
    @in_file_path = args[0]
    @out_file_path = args[1]
  end

  def run
    # Business logic goes here
  end
end

# Use Ruby constants to make the file runnable from the command line
if $PROGRAM_NAME == __FILE__
  AddNumbers.new(ARGV).run
end
{% endhighlight %}

Now it's easy to call the script from a spec. We've parameterized the path of the input file and the path of the output file, we've asked the caller of the class to supply the array of arguments rather than using the constant `ARGV`, and we've retained the ability to run the script from the command line using Ruby constants. To call the script from a spec, we now simply need to do the following:

{% highlight ruby %}
require_relative './add_numbers.rb'
require 'rspec'

RSpec.describe AddNumbers do
  subject { described_class.new(arguments).run }

  let(:arguments) { ['test.csv', 'test_out.csv'] }

  it 'runs' do
    subject
  end
end
{% endhighlight %}

Filling out the business logic gives us:

{% highlight ruby %}
#!/usr/bin/env ruby
require 'csv'

class AddNumbers
  attr_accessor :in_file_path, :out_file_path

  def initialize(args)
    @in_file_path = args[0]
    @out_file_path = args[1]
  end

  def run
    number_pairs.each do |pair|
      out_file << [pair[0].to_i + pair[1].to_i]
    end

    out_file.close
  end

  private

  def number_pairs
    @number_pairs ||= CSV.readlines(in_file_path)
  end

  def out_file
    @out_file ||= CSV.open(out_file_path, 'wb')
  end
end

# Use Ruby constants to make the file runnable from the command line
if $PROGRAM_NAME == __FILE__
  AddNumbers.new(ARGV).run
end
{% endhighlight %}

## Faking the Files

Running the above spec fails because `test.csv` doesn't exist. We need some way of providing a fake file to the spec. There are numerous ways to do this, from using `double` to mock the filesystem, to including a gem that mocks the filesystem completely. The best way I've found, however, is to use actual files that live only for the duration of the spec. Ruby's [Tempfile][tempfile-doc] is perfect for this.

For the output file, all we need to do is create a blank Tempfile:

{% highlight ruby %}
let(:test_out_file) { Tempfile.new('csv') }
{% endhighlight %}

For the input file, we need to write pairs of numbers to it when it is created. We can call `tap` on the file when it's created and write rows of numbers to it in the spec:

{% highlight ruby %}
let(:test_in_file) do
  Tempfile.new('csv').tap do |f|
    pairs.each do |pair|
      f << pair.join(',') + "\r"
    end

    f.close
  end
end

let(:pairs) do
  [
    [10,10],
    [20,40],
    [30,50]
  ]
end
{% endhighlight %}

As the Tempfile documentation suggests, we call `unlink` on the files after each spec to ensure that the files get deleted.

{% highlight ruby %}
after do
  test_in_file.unlink
  test_out_file.unlink
end
{% endhighlight %}

And that's all we need to test the script on arbitrary sets of numbers! The complete spec with one example is below:

{% highlight ruby %}
require_relative './add_numbers.rb'
require 'rspec'
require 'tempfile'

RSpec.describe AddNumbers do
  subject { described_class.new(arguments).run }

  let(:arguments) { [test_in_file.path, test_out_file.path] }

  let(:test_out_file) { Tempfile.new('csv') }
  let(:test_in_file) do
    Tempfile.new('csv').tap do |f|
      pairs.each do |pair|
        f << pair.join(',') + "\r"
      end

      f.close
    end
  end

  let(:pairs) do
    [
      [10,10],
      [20,40],
      [30,50]
    ]
  end

  after do
    test_in_file.unlink
    test_out_file.unlink
  end

  it 'writes the sums to a file' do
    subject
    expect(CSV.open(test_out_file.path).readlines).to eq(
      [["20"], ["60"], ["80"]]
    )
  end
end
{% endhighlight %}

## Testing the Edge Cases

What was the point of writing all of that setup if we're only going to have one example? Using `let` to set up the spec variables makes it easy to test out the edge cases of our script. For an example, we'll add a pair of numbers where one is negative to make sure the script still works:


{% highlight ruby %}
context 'with negative numbers in the input file' do
  let(:pairs) { super() << [-9, 20] }

  it 'writes the sums to a file' do
    subject
    expect(CSV.open(test_out_file.path).readlines).to eq(
      [["20"], ["60"], ["80"], ["11"]]
    )
  end
end
{% endhighlight %}

Luckily, it still works. And we can be confident that it will as long as the specs continue to pass.

[tempfile-doc]: http://ruby-doc.org/stdlib-2.3.0/libdoc/tempfile/rdoc/Tempfile.html
