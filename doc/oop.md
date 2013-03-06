# 他言語学習者のための Perl オブジェクト指向プログラミング入門

筆者の母国語は Ruby なので、基本的には Ruby の用語にのっとって書きますが、Java や PHP もかんたんに通ってるのでその辺の人が読んでも不都合はないものになる予定です。

上記からわかるとおり、ここで解説しようとしているオブジェクト指向プログラミングとはクラスベースのそれを指します。

TMTOWTDI の Perl らしく、ここに書くやり方以外にもクラスベースの OOP を実装する手段はありますが、一つの実装例として読んでください。

## まずはコードから。

実装されたコードに勝るサンプルはないということで、お手本を読むところから始めましょう。

Perl できれいに実装された OOP のコードを見るなら Plack がおすすめです。

https://github.com/plack/Plack/blob/master/lib/Plack/Request.pm

## クラス宣言

まずはクラス宣言をしなければ始まりません。クラスの宣言は package を使って行います。

```perl
package Plack::Request;
use strict;
use warnings;

```

`use strict;` や `use warnings;` は Perl モジュールのおまじないなので、初めてみる方は別途調べてください。

結果としてクラス宣言として働きますが、これは単なるモジュール化の手段としての package 文です。package として扱われるスコープなども通常の package 文の利用と変わりありません。

## コンストラクタ

ただの package 文がクラスとして働くためには、オブジェクト化が必要です。Perl でのコンストラクタは new というサブルーチンを作成します。

```perl
sub new {
    my($class, $env) = @_;
    Carp::croak(q{$env is required})
        unless defined $env && ref($env) eq 'HASH';

    bless { env => $env }, $class;
}
```

`my($class, $env) = @_;` のように、第一引数としてクラス名 (package 名) を受け取ります。オブジェクト化のトリックは `bless { env => $env}, $class;` にあります。

`{ env => $env }` というハッシュリファレンスを $class を引数に bless することで、$class パッケージのサブルーチンを呼び出すことができるリファレンスができあがります。

こうしてできたハッシュリファレンスはこのように使えます。

```perl
my $req = Plack::Request->new();
print $req->remote_host();
```

$req は Plack::Request のインスタンスとして振る舞いますが、実際は new サブルーチンの結果として返る bless されたハッシュリファレンスです。new を使うのは慣例であり、Perl のシンタクスではありません。

ここでは、`{PackageName}->(sub}()` (`{sub} {PackageName}()` とも書けます) という呼び出しをするとパッケージ名が暗黙の第一引数としてわたされる、というのが Perl のシンタクスとしての機能です (PackageName::sub でも静的なサブルーチンとしての呼び出しは可能です。ただし、この場合は継承木はたどらず、静的なパッケージ関数の呼出しとして解釈され sub の引数もパッケージ名はわたされません)。

コンストラクタの中でインスタンス自身を操作したいときは、このようにも書けます。便宜上コンストラクタと呼びましたが、実際はたんなる Factory Method なので、最後に bless されたハッシュリファレンスを return してやればいいだけです。

```perl
sub new {
    my $class = shift;
    my $self = {};
    bless $self, $class;
    # do something
    return $self;
}
```

## インスタンスメソッド

メソッドはパッケージのサブルーチンとして定義します。インスタンス (bless されたハッシュリファレンス) から呼び出されるとインスタンスメソッドとして振る舞います。

```perl
sub headers {
    my $self = shift;
    if (!defined $self->{headers}) {
        my $env = $self->env;
        $self->{headers} = HTTP::Headers->new(
            map {
                (my $field = $_) =~ s/^HTTPS?_//;
                ( $field => $env->{$_} );
            }
                grep { /^(?:HTTP|CONTENT)/i } keys %$env
            );
    }
    $self->{headers};
}

```

インスタンスから -> により呼び出されたサブルーチン (インスタンスメソッド) は第一引数に自身を表すリファレンスを受け取ります。慣例として $self と名付けて受け取ります。

## インスタンス変数

インスタンス変数は bless されたハッシュリファレンスの要素を使います。インスタンスメソッドで受けとるインスタンス $self は実際には bless されたハッシュリファレンスなので、ハッシュリファレンスとしてのアクセスもできます。結果として、下記のようにハッシュの要素をインスタンス変数のように扱うことができます。

`$self->{middlewares}` はインスタンス変数のためのシンタクスではなく、$self ハッシュリファレンスに対する要素の操作です。

[Plack::Builder](https://github.com/plack/Plack/blob/master/lib/Plack/Builder.pm) の例。

```perl
sub new {
    my $class = shift;
    bless { middlewares => [ ] }, $class;
}

sub add_middleware {
    my($self, $mw, @args) = @_;

    if (ref $mw ne 'CODE') {
        my $mw_class = Plack::Util::load_class($mw, 'Plack::Middleware');
        $mw = sub { $mw_class->wrap($_[0], @args) };
    }

    push @{$self->{middlewares}}, $mw;
}

```

## private メソッド

private なメソッドには、sub _private_method というような命名をします。言語の機能としてアクセシビリティの制御をするわけではなく、紳士協定として外向けの API として公開しませんよという意思表示をする慣例です。

不安に思うかもしれませんが、Ruby の private メソッドもシンタクスはありますが迂遠な呼び方をすれば呼ぶことはでき、実際のところは[「外から呼び出しにくくする」という設計](http://blade.nagaokaut.ac.jp/cgi-bin/scat.rb/ruby/ruby-list/47615)なのでその意味で Perl の private メソッドも同じような理解でいいでしょう。

public なメソッドは sub public_method というように _ をつけません。

自分の足を打ったときは血を流しつつも前進するのが Perl (偏見) なので Java とは違うのです:p

## クラス変数

クラス変数は Perl の package 変数です。public なものは our を使って定義します。private なものは my を使って定義します (package はブロックスコープをつくらないので一つのファイルに複数 package を定義しているようなときは注意)。

## クラスメソッド

クラスメソッドは package のサブルーチンとして定義し、`{PackageName}->{sub}()` で呼び出します。このときサブルーチンの第一引数としてパッケージ名が渡されます。つまり、コンストラクタと呼んでいた new はただのクラスメソッドです。

[Plack::MIME](https://github.com/plack/Plack/blob/master/lib/Plack/MIME.pm) での例です。

```perl
package Plack::MIME;
use strict;
~

sub mime_type {
    my($class, $file) = @_;
    $file =~ /(\.[a-zA-Z0-9]+)$/ or return;
    $MIME_TYPES->{lc $1} || $fallback->(lc $1);
}

~
=head1 SYNOPSIS

  use Plack::MIME;

  my $mime = Plack::MIME->mime_type(".png"); # image/png
```

## 定数

OOP 以前に Perl としても[諸説](http://d.hatena.ne.jp/fbis/20090612/1244806476)あるところなので、むずかしい...

constant プラグマでもいいのではないですかねと思う PHP の constant で苦労した人なのでした。

## 継承

`use parent` を使います。Perl の継承は多重継承です。

[Amon2::Web::Request](https://github.com/tokuhirom/Amon/blob/master/lib/Amon2/Web/Request.pm) での例。

```perl
package Amon2::Web::Request;
use strict;
use warnings;
use parent qw/Plack::Request/;
```

親クラスのメソッドを呼び出すときは `SUPER::` 経由で呼び出します。

クラスメソッドの場合。

```perl
sub new {
    my ($class, $env, $context_class) = @_;
    my $self = $class->SUPER::new($env);
    if (@_==3) {
        $self->{_web_pkg} = $context_class;
    }
    return $self;
}
```

インスタンスメソッドの場合。

```perl
sub body_parameters {
    my ($self) = @_;
    $self->{'amon2.body_parameters'} ||= $self->_decode_parameters($self->SUPER::body_parameters());
}

```

use base というプラグマもありましたが、["Don't use base.pm, use parent.pm instead!"](http://d.hatena.ne.jp/gfx/20101226/1293342019) とのことなので parent を使いましょう。

## DSL を export

[Plack::Builder](https://github.com/plack/Plack/blob/master/lib/Plack/Builder.pm) が参考になります。

Export を使いグローバルの名前空間を使うので、利用には注意が必要です。

```perl
package Plack::Builder;
use strict;
use parent qw( Exporter );
our @EXPORT = qw( builder add enable enable_if mount );

~

# DSL goes here
our $_add = our $_add_if = our $_mount = sub {
    Carp::croak("enable/mount should be called inside builder {} block");
};

sub enable         { $_add->(@_) }
sub enable_if(&$@) { $_add_if->(@_) }

sub mount {
    my $self = shift;
    if (Scalar::Util::blessed($self)) {
        $self->_mount(@_);
    }else{
        $_mount->($self, @_);
    }
}

sub builder(&) {
    my $block = shift;

    my $self = __PACKAGE__->new;

    my $mount_is_called;
    my $urlmap = Plack::App::URLMap->new;
    local $_mount = sub {
        $mount_is_called++;
        $urlmap->map(@_);
        $urlmap;
    };
    local $_add = sub {
        $self->add_middleware(@_);
    };
    local $_add_if = sub {
        $self->add_middleware_if(@_);
    };

    my $app = $block->();

    if ($mount_is_called) {
        if ($app ne $urlmap) {
            Carp::carp("WARNING: You used mount() in a builder block, but the last line (app) isn't using mount().\n" .
                       "WARNING: This causes all mount() mappings to be ignored.\n");
        } else {
            $app = $app->to_app;
        }
    }

    $app = $app->to_app if $app and Scalar::Util::blessed($app) and $app->can('to_app');

    $self->to_app($app);
}

~

=head1 SYNOPSIS

  # in .psgi
  use Plack::Builder;

  my $app = sub { ... };

  builder {
      enable "Deflater";
      enable "Session", store => "File";
      enable "Debug", panels => [ qw(DBITrace Memory Timer) ];
      enable "+My::Plack::Middleware";
      $app;
  };

  # use URLMap

  builder {
      mount "/foo" => builder {
          enable "Foo";
          $app;
      };

      mount "/bar" => $app2;
      mount "http://example.com/" => builder { $app3 };
  };
```

## 他言語プログラマのための my, our, local の解説

Perl は元がすべてグローバル変数。use strict 配下だとグローバル変数が抑止されます。そこで、ブロックスコープのグローバル変数を宣言するのが my 、モジュールスコープのグローバル変数を宣言するのが our 。local は寿命がブロックの終わりまでで、スコープはグローバルスコープとなります。

our か my かはスコープの違いを表現していて、package レベルで宣言された時点でクラスにしろモジュールにしろ自動的に寿命はプロセスの生存期間中になります。

