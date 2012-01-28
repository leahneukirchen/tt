#!/usr/bin/env ruby
# tt - a 9term-compatible terminal in Ruby/Tk.

# Written by Christian Neukirchen <chneukirchen@gmail.com> in 2012.
# To the extent possible under law, Christian Neukirchen has waived
# all copyright and related or neighboring rights to this work.
# http://creativecommons.org/publicdomain/zero/1.0/

require 'tk'
require 'pty'

Thread.abort_on_exception = true

SIGINT = 2
case RUBY_PLATFORM
when /linux/
  TIOCSWINSZ = 0x5414
  TIOCSIG    = 0x40045436
else
  TIOCSWINSZ = nil
  TIOCSIG    = nil
end

DEFAULT_FONT = '-misc-fixed-bold-r-normal--15-140-75-75-c-90-iso8859-1'  # 9x15bold
ALTERNATE_FONT = '-b&h-lucida-medium-r-normal-*-12-120-75-75-p-71-iso10646-1'

class TT
  VERSION = '0.1'

  def initialize(opts={})
    @opts = opts

    @root = TkRoot.new {
      geometry(opts["geom"])  if opts["geom"]
    }
    self.title = opts["title"]
    @root.bind("Map") { @mapped = true; self.title = self.title }
    @root.bind("Unmap") { @mapped = false }

    if @opts["sb"]
      @yscroll = TkScrollbar.new(@root) {
        pack 'side' => 'left', 'fill' => 'y'
      }
    end

    @scroll = TkVariable.new
    @scroll.value = opts["sc"]
    @mono = TkVariable.new
    @mono.value = opts["fw"]

    @panes = TkPanedwindow.new(@root) {
      orient 'vertical'
    }

    @bar = TkText.new(@panes) {
      height opts["bar"].to_s.count("\n") + 1
      foreground opts["fg"]
      background opts["bg"]
      insertbackground opts["cr"] || opts["fg"]
      insertofftime 0  # don't blink
      tabstyle 'wordprocessor'
      pack 'fill' => 'both', 'expand' => 'yes'
    }

    @bar.insert 'end', opts["bar"]

    @bar.tag_configure 'run', 'underline' => true

    @bar.bind('B2-Motion') { |ev|
      @bar.tag_remove 'run', '0.0', 'end'
      @bar.tag_add 'run', 'run', "@#{ev.x},#{ev.y}"
      @bar.tag_add 'run', "@#{ev.x},#{ev.y}", 'run'
      Tk.callback_break
    }

    @bar.bind('ButtonPress-2') { |ev|
      @bar.tag_remove 'run', '0.0', 'end'
      @bar.mark_set 'run', "@#{ev.x},#{ev.y}"
    }

    @bar.bind('ButtonRelease-2') { |ev|
      a, b = @bar.tag_ranges('run').first
      if a.nil?  # use single word
        cmd = @bar.get("@#{ev.x},#{ev.y} wordstart",
                       "@#{ev.x},#{ev.y} wordend")
      else
        cmd = @bar.get(a, b)
      end

      bar_cmd cmd

      @bar.tag_remove 'run', '0.0', 'end'
      Tk.callback_break
    }

    # XXX provide text via socket?
    @text = TkText.new(@panes) {
      yscrollbar @yscroll  if opts["sb"]
      height opts["height"]
      width opts["width"]
      foreground opts["fg"]
      background opts["bg"]
      insertbackground opts["cr"] || opts["fg"]
      insertofftime 0  # don't blink
      tabstyle 'wordprocessor'
      pack 'fill' => 'both', 'expand' => 'yes'
    }

    @text.bind("KeyPress") { |o|
      case o.keysym
      when "Next"
        # Put cursor at end on last Next.
        if @text.yview[1] > 0.99999
          @text.mark_set 'insert', 'end'
        end
        # PASSTHRU, no callback_break!
      when "Prior", "Up", "Down", "Left", "Right"
        # XXX good idea left/right?
        # PASSTHRU, no callback_break!
      when "Home"
        @text.yview_moveto 0
        @text.mark_set('insert', '1.0')
        Tk.callback_break
      when "End"
        @text.yview_moveto 1
        @text.mark_set('insert', 'end')
        Tk.callback_break
      else
        # are we at the end?
        if @text.index('end') == @text.index('insert + 1 chars')
          @output.write o.char
          @text.see('end')
          @text.mark_set('lastinsert', 'insert')
          Tk.callback_break
        elsif o.char == ?\C-w
          # XXX skip whitespace first
          @text.delete('insert - 1c wordstart', 'insert')
        elsif o.char == ?\C-u
          @text.delete('insert linestart', 'insert')
        else
          # PASSTHRU, no callback_break!
        end
      end

    }

    @text.bind('B2-Motion') { |ev|
      @text.tag_remove 'run', '0.0', 'end'
      @text.tag_add 'run', 'run', "@#{ev.x},#{ev.y}"
      @text.tag_add 'run', "@#{ev.x},#{ev.y}", 'run'
      Tk.callback_break
    }

    @text.bind('ButtonPress-2') { |ev|
      @text.tag_remove 'run', '0.0', 'end'
      @text.mark_set 'run', "@#{ev.x},#{ev.y}"
    }

    @text.bind('ButtonRelease-2') { |ev|
      a, b = @text.tag_ranges('run').first
      if a.nil?  # no selection
        paste_callback(nil)
      else
        cmd = @text.get(a, b)
      end

      bar_cmd cmd

      @text.tag_remove 'run', '0.0', 'end'
      Tk.callback_break
    }

    @text.tag_configure "href", :underline => true
    @text.tag_configure "run", :underline => true

    @text.mark_set 'lastinsert', '1.0'
    @text.mark_gravity 'lastinsert', 'left'

    @hold = TkText.new(@panes) {
      foreground opts["fg"]
      background opts["bg"]
      insertbackground opts["cr"] || opts["fg"]
      insertofftime 0  # don't blink
      tabstyle 'wordprocessor'
      pack 'fill' => 'both', 'expand' => 'yes'
    }
    @hold.bind("Shift-Return") {
      @text.mark_set('insert', 'end')
      @output.write @hold.get('1.0', 'end')
      @text.mark_set('insert', 'end')
      @text.mark_set('lastinsert', 'insert')
      @hold.delete('1.0', 'end')
      Tk.callback_break
    }

    if opts["bar"]
      @panes.add @bar, :stretch => 'never'
    else
      @panes.add @bar, :stretch => 'never', :height => 1
    end
    @panes.add @text, :stretch => 'always'
    @panes.add @hold, :stretch => 'never', :height => 1
    @panes.pack 'fill' => 'both', 'expand' => 'yes'

    font_cmd

    menu = TkMenu.new(@root) { tearoff false }

    menu.add_command :label => "fwd", :command => method(:fwd_cmd)
    menu.add_command :label => "bwd", :command => method(:bwd_cmd)
    menu.add_command :label => "cut", :command => method(:cut_cmd)
    menu.add_command :label => "paste", :command => method(:paste_cmd)
    menu.add_command :label => "send", :command => method(:send_cmd)
    menu.add_command :label => "clear", :command => method(:clear_cmd)
    menu.add_checkbutton :label => "mono", :variable => @mono, :command => method(:font_cmd)
    menu.add_checkbutton :label => "scroll", :variable => @scroll
    @lastcmd = 0

    # hacky
    menufontheight = TkFont.metrics("TkMenuFont", "linespace")+5
    @root.bind 'Button-3', lambda { |ev|
      menu.popup(ev.x_root-10,ev.y_root-10-@lastcmd*menufontheight)
    }

    if TIOCSWINSZ
      pw = ph = 0
      timer = nil
      @root.bind('Configure') { |o|
        if @pid
          unless pw == o.width && ph == o.height
            pw = o.width
            ph = o.height

            # wait 500ms before setting terminal size to avoid stutter
            timer ||= TkTimer.start(500) {
              w = @text.winfo_width / @font_width
              h = @text.winfo_height / @font_height

              @output.ioctl(TIOCSWINSZ, [h, w, ph, pw].pack('SSSS'))

              timer.cancel
              timer = nil
            }
          end
        end
      }
    end

    @text.focus
  end

  def paste_callback(event)
    sel = TkSelection.get(:type => "UTF8_STRING")  rescue ""
    @text.mark_set('insert', 'end')
    @output.write sel
    @text.mark_set('insert', 'end')
    @text.mark_set('lastinsert', 'insert')
    Tk.callback_break
  end

  attr_reader :font
  def font=(font)
    @bar.font = font
    @text.font = font
    @hold.font = font
    @font = font
    @font_height = TkFont.metrics(font, "linespace")
    @font_width = TkFont.measure(font, "0")
  end

  attr_reader :title
  def title=(title)
    if @opts["zi"] && !@mapped
      if @oldtitle != "*** #{title}"
        @root.title = "*** #{title}"
      end
    else
      if @oldtitle != title
        @root.title = title
      end
    end
    @oldtitle = @root.title
    @title = title
  end

  # hack
  def fix_utf8(buf)
    buf.force_encoding("UTF-8")
    buf.encode!("UTF-16", :invalid => :replace, :undef => :replace)
    buf.encode!("UTF-8")
    buf
  end

  URL_RE = %r{((?:https?://|ftp://|news://|mailto:|file://|\bwww\.)[a-zA-Z0-9\-\@;\/?:&=%\$_.+!*\x27,~#]*(?:\([a-zA-Z0-9\-\@;\/?:&=%\$_.+!*\x27,~#]*\)|[a-zA-Z0-9\-\@;\/?:&=%\$_+*~])+)}

  def run(command=nil)
    command ||= @opts["cmd"]
    ENV["TERM"] = @opts["tn"]
    ENV["WINDOWID"] = (Integer(Tk::Wm.frame(@root)) + 1).to_s  # ugh
    @input, @output, @pid = PTY.spawn(*command)

    Thread.new {
      begin
        while buf = @input.readpartial(4096)
          # Try to sanitize CRLF.
          buf = fix_utf8(buf).gsub(/\r+\n/, "\n").gsub(/\r {10,}\r/, "")

          out = ""
          buf.each_char { |char|
            if char == "\b"
              if out.empty?
                @text.delete('end - 2 chars')
              else
                out.chop!
              end
            elsif char == "\r"
              if out.empty?
                # delete to beginning of line
                @text.delete('end - 1c linestart', 'end - 1c')
              end
              # else ignore them
            else
              out << char
            end
          }

          out.gsub!(/\033\];(.*?)\007/) {
            self.title = $1
            ""
          }

          if @opts["cu"]
            out.split(URL_RE).each { |piece|
              if piece =~ URL_RE
                @text.insert 'end', piece, "href"
                @text.tag_bind "href", "1", method(:link_action)
              else
                @text.insert 'end', piece
              end
            }
          else
            @text.insert 'end', out
          end

          self.title = self.title

          if @scroll.value == "1"
            @text.yview_moveto 1
          else
            @text.yview 'lastinsert'
          end
        end
      rescue Errno::EIO
        # XXX too late now anyway?
        exit
      end
    }
  end

  def fwd_cmd
    @lastcmd = 0
    sel = @text.get('sel.first', 'sel.last')  rescue return
    if match = @text.tksearch(sel, 'sel.last', 'end')
      @text.tag_remove 'sel', 'sel.first', 'sel.last'
      @text.tag_add 'sel', match, "#{match} + #{sel.size} chars"
      @text.mark_set('insert', 'sel.first')
      @text.see('insert')
    end
  end

  def bwd_cmd
    @lastcmd = 1
    sel = @text.get('sel.first', 'sel.last')  rescue return
    if match = @text.tksearch(['backwards'], sel, 'sel.first', '1.0')
      @text.tag_remove 'sel', 'sel.first', 'sel.last'
      @text.tag_add 'sel', match, "#{match} + #{sel.size} chars"
      @text.mark_set('insert', 'sel.first')
      @text.see('insert')
    end
  end

  def cut_cmd
    @lastcmd = 2
    sel = @text.get('sel.first', 'sel.last')  rescue return
    @text.delete('sel.first', 'sel.last')

    TkSelection.clear
    TkSelection.handle(Tk.root, :selection => "PRIMARY",
                       :type => "UTF8_STRING") { sel }
    TkSelection.set_owner(Tk.root, :selection => "PRIMARY")
  end

  def paste_cmd
    @lastcmd = 3
    sel = TkSelection.get(:type => "UTF8_STRING")  rescue ""
    @text.insert('insert', sel)
  end

  def send_cmd
    @lastcmd = 4
    sel = @text.get('sel.first', 'sel.last')  rescue return
    sel << "\n"  unless sel[-1] == "\n"
    @text.mark_set('insert', 'end')
    @output.write sel
    @text.mark_set('insert', 'end')
    @text.mark_set('lastinsert', 'insert')
  end

  def clear_cmd
    @lastcmd = 5
    @text.delete('1.0', 'end')
  end

  def font_cmd
    if @mono.value == "1"
      self.font = @opts["fn"]
    else
      self.font = @opts["af"]
    end
  end

  def bar_cmd(cmd)
    case cmd.lstrip
    when "Font"
      @mono.value = 1 - @mono.value.to_i
      font_cmd
    when /\AFont /
      begin
        self.font = $'
      rescue
      end
    when "Scroll"
      @scroll.value = 1 - @scroll.value.to_i
    when "Paste"
      paste_cmd
    when "Cut"
      cut_cmd
    when "Fwd"
      fwd_cmd
    when "Bwd"
      bwd_cmd
    when "Send"
      send_cmd
    when "Clear"
      clear_cmd
    when "Intr"
      @input.ioctl(TIOCSIG, 2)
    when /\A(\/|Fwd )/
      if match = @text.tksearch($', 'insert + 1c', 'end')
        @text.tag_remove 'sel', 'sel.first', 'sel.last'  rescue nil
        @text.tag_add 'sel', match, "#{match} + #{$'.size} chars"
        @text.mark_set('insert', 'sel.first')
        @text.see('insert')
      end
    when /\A(\?|Bwd )/
      if match = @text.tksearch(['backwards'], $', 'insert', '1.0')
        @text.tag_remove 'sel', 'sel.first', 'sel.last'  rescue nil
        @text.tag_add 'sel', match, "#{match} + #{$'.size} chars"
        @text.mark_set('insert', 'sel.first')
        @text.see('insert')
      end
    when "Quit"
      exit
    else
      @output.write cmd + "\n"
      @text.mark_set('insert', 'end')
      @text.mark_set('lastinsert', 'insert')
    end
  end

  def link_action(ev)
    a, b = @text.tag_prevrange("href", @text.index("@#{ev.x},#{ev.y}"))
    url = @text.get(a, b)
    system @opts["browser"], url
  end
end

opts = {
  "sb" => true,
  "fw" => true,
  "sc" => true,
  "cu" => true,
  "zi" => true,
  "bg" => "#ffffff",
  "fg" => "#000000",
  "fn" => DEFAULT_FONT,
  "af" => ALTERNATE_FONT,
  "browser" => "xdg-open",
  "width" => "80",
  "height" => "24",
  "tn" => "9term",
  "title" => "tt",
  "cmd" => [ENV["SHELL"] || "/bin/sh"],
  "bar" => nil,
}
until ARGV.empty?
  case arg = ARGV.shift
  when /\A([+-])(sb|sc|fw|cu|zi)\z/
    opts[$2] = ($1 == "-")
  when /\A-(fn|af|tn|fg|bg|cr|title|browser|bar)\z/
    opts[$1] = ARGV.shift
  when /\A-(g(eometry)?)\z/
    g = ARGV.shift
    if g =~ /\A(\d+x\d+)?([+-]\d+[+-]\d+)?\z/
      opts["width"], opts["height"] = $1.split("x")  if $1
      opts["geom"] = $2
    else
      abort "invalid geometry #{g}"
    end
  when "-e"
    opts["cmd"] = ARGV
    break
  when "-h", "-help", "--help"
    puts "tt #{TT::VERSION}"
    puts "Usage: tt [-help] [--help]
 [-tn string] [-geometry geometry] [-/+fw] [-/+sb] [-/+sc] [-/+zi]
 [-bg color] [-fg color] [-cr color] [-fn fontname] [-af fontname]
 [-bar string] [-browser string] [-title string] [-e command arg...]"
    exit
  else
    abort "invalid argument #{arg}"
  end
end

t = TT.new(opts)
t.run
Tk.mainloop
