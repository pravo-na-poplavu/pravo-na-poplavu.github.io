=begin
TODO:

красивість:
  пошук: стиль форми хоч трохи?
  ?стиль «клацалок» та інших елементів

v2:
  ?теги? автоматично витягнути усі слова; накидати фільтрів, напівавтоматично згрупувати по темах
  toc та форматування сторінок: гості окремо, тема (якщо є) окремо, теги/теми?
    у toc підказка з усіма питаннями
  https://github.com/o-oconnell/mp4grep → https://alphacephei.com/vosk/models
    ffmpeg -loglevel quiet -i poplava60.mp4 -ar 16000 -ac 1 -f s16le poplava60_16.mp4
=end

require 'yaml'
require 'backports/latest'
require 'active_support/all'

class Array
  def to_proc
    proc { _1[*self] }
  end
end

def parse_time(str)
  str.split(':').reduce(0) { |sum, n| sum * 60 + n.sub(/^0/, '').to_i }
end

def render_date(dt, short: false)
  months = short ? %w[- січ лют бер кві тра чер лип сер вер жов лис гру] :
                   %w[- січня лютого березня квітня травня червня липня серпня вересня жовтня листопада грудня]
  '%i %s' % [dt.day, months[dt.month]]
end

file '_data/videos_raw.yml' do |t|
  require 'yt'
  require 'dotenv'
  Dotenv.load!

  Yt.configuration.api_key = ENV.fetch('YOUTUBE_API_KEY')
  Yt.configuration.log_level = :debug

  channel = Yt::Channel.new(id: 'UCwCkRo2WQx_9JRWISLC47fw')

  # TODO: витягати тількі свіжі
  channel.videos.map { |ref|
    # Channel#videos витягнув зі скороченими descriptions. FIXME: може є опція?
    video = Yt::Video.new(id: ref.id)

    %i[id title length description published_at tags].to_h { [_1, video.send(_1)] }
  }.then { File.write t.name, _1.to_yaml }
end

file '_data/videos.yml' => '_data/videos_raw.yml' do |t|
  YAML
    .load_file(t.prerequisites.first)
    .select { _1[:title].match?(%r{#\d+ фронтова поплава}i) }
    .map { |video|
      puts "Обробка #{video[:title]}"

      num = video[:title][/^\#(\d+)/, 1].to_i

      if (pos = video[:description].match(/\n(\d+:\d+.+)\z/m)&.begin(1))
        fragments =
          video[:description][pos..]
          .split("\n")
          .map { |ln| ln.scan(/^(\d+:\d+(?::\d+)?)[- —]+(.+)$/).flatten }
          .each { _1.length == 2 or raise "Weird timecode line: #{_1}"}
          .map { %i[start title].zip(_1).to_h }
          .tap { |ary| ary.each_cons(2) { _1[:end] = _2[:start] } }
          .tap { _1.last[:end] = video[:length] }
          .map { _1.merge(start_seconds: parse_time(_1[:start]), end_seconds: parse_time(_1[:end])) }
        video[:description] = video[:description][..pos] # викинути їх з дескріпшена
      else
        # Таймкодів немає для поплав 5-8
        fragments = []
      end

      video.except(:id).merge(
        video_id: video[:id],
        fragments: fragments,
        number: num,
        number_str: '%02i' % num,
        short_title: video[:title].sub(' Фронтова поплава', ''),
        short_date: render_date(video[:published_at], short: true),
        date: render_date(video[:published_at])
      )
    }.tap { |videos|
      videos.each_cons(2) {
        _1[:next] = _2.slice(:number_str, :title)
        _2[:previous] = _1.slice(:number_str, :title)
      }
    }
    .then { File.write t.name, _1.map(&:deep_stringify_keys).to_yaml }
end

require 'memoist'

extend Memoist

memoize def videos_data
  YAML.load_file('_data/videos.yml')
end

rule %r{videos/\d+\.md} => '_data/videos.yml' do |r|
  num = r.name[/\d+/].to_i
  video = videos_data.find { _1['number'] == num }

  File.write(r.name, video.merge('layout' => 'video').to_yaml + "---\n")
end

file '_data/fragments.json' => '_data/videos.yml' do |t|
  videos_data.map(&:deep_symbolize_keys).flat_map { |video|
    video[:fragments].map {
      _1.merge(
        video: video.except(:description, :fragments),
        id: "#{video[:video_id]}@#{_1[:start_seconds]}"
      )
    }
  }.then { File.write(t.name, _1.to_json) }
end

file '_data/index.json' => '_data/fragments.json' do |t|
  `cat #{t.prerequisites.first} | node _script/build-index.js > #{t.name}`
end

file 'search.md' => %w[_data/fragments.json _data/index.json] do |t|
  File.write(t.name,
             {
               title: 'Пошук',
               layout: 'search',
               fragments: File.read(t.prerequisites.first),
               index: File.read(t.prerequisites.last)
             }.deep_stringify_keys.to_yaml +
             "---\n")
end

desc 'Витягнути інфу по відео з ютюба'
task fetch_videos: '_data/videos_raw.yml'

desc 'Розбити та причесати описи відео'
task parse_videos: '_data/videos.yml'

# TODO: автоматично список сторінок!
desc "Згенерувати сторінки з відео"
task video_pages: (1..68).map { 'videos/%02i.md' % _1 }

desc "Згенерувати пошуковий індекс"
task search_page: 'search.md'

task default: %i[video_pages search_page]
