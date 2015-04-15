# PhpSpec Learning

* 如何開發 Composer-based library ？
* 如何從預期的結果往回推出程式碼？
* 如何用更語義化的方式撰寫測試？

## 初始化專案

```bash
composer require phpspec/phpspec
```

```bash
alias t=./vendor/bin/phpspec
```

編輯 `phpspec.yml`

```yaml
suites:
  main:
    namespace: KK
    psr4_prefix: KK
```

編輯 `composer.json`

```json
  "autoload": {
    "psr-4": {
      "KK\\": "src/"
    }
  }
```

```bash
composer dump
```

## 範例說明

主角：

* 播放清單 (Playlist)
* 歌曲 (Song)

播放清單規格：

* 可以加入單首歌曲
* 可以一次加入多首歌曲
* 可以將清單內所有歌曲設定為已播放

歌曲規格：

* 可以加星
* 不可加超過 5 的星
* 可以被設定為已播放
* 可以取得歌曲名稱

## 建立播放清單規格類別

```bash
t desc KK/Playlist
```

```bash
t run
```

```
Do you want me to create `KK\Playlist` for you? (y)
```

* PhpSpec 會自動幫我們建立對應的檔案

### 規格一：可以加入單首歌曲

編輯 `spec/PlaylistSpec.php`

```php
use KK\Song;

    function it_add_a_song_to_playlist(Song $song)
    {
        $this->add($song);
        $this->shouldHaveCount(1);
    }
```

* 用 Double 來隔離 `Song` 類別，因為我們還沒實作
* PhpSpec 會自動注入 Double 物件

```bash
t run
```

```
Class spec\KK\Song does not exist
```

* PhpSpec 無法自動生成 Double 物件的類別

編輯 `src/Song.php`

```php
namespace KK;

class Song
{
}
```

```bash
t run
```

```
Do you want me to create `KK\Playlist::add()` for you? (y)
```

* PhpSpec 會自動幫我們建立對應的方法

```
Do you want me to create `KK\Playlist::hasCount()` for you? (n)
```

* `Playlist` 類別要實作 `Countable` 介面，所以不新增 `hasCount` 方法

編輯 `src/Playlist.php`

```php
namespace KK;

use Countable;

class Playlist implements Countable
{
    protected $songs;

    public function add($song)
    {
        $this->songs[] = $song;
    }

    public function count()
    {
        return count($this->songs);
    }
}
```

* `Playlist` 內部用陣列來存放新增的歌曲

```bash
t run
```

### 規格二：可以一次加入多首歌曲

編輯 `spec/PlaylistSpec.php`

```php
    function it_can_accept_multiple_songs_to_add_at_once(Song $song1, Song $song2)
    {
        $this->add([$song1, $song2]);
        $this->shouldHaveCount(2);
    }
```

```bash
t run
```

編輯 `src/Playlist.php`

```php
    public function add($song)
    {
        if (is_array($song)) {
            return array_map([$this, 'add'], $song);
        }

        $this->songs[] = $song;
    }
```

* 利用 `array_map` 與 callback 來實作

```bash
t run
```

## 引入 Mock 物件

編輯 `spec/PlaylistSpec.php`

```php
    function it_can_mark_all_songs_as_played(Song $song1, Song $song2)
    {
        $song1->play()->shouldBeCalled();
        $song2->play()->shouldBeCalled();

        $this->add([$song1, $song2]);
        $this->markAllAsPlayed();
    }
```

* Double 物件被預期某方法可能會被呼叫時，就是 Mock 物件

```bash
t run
```

```
method `Double\KK\Song\P4::play()` is not defined.
```

* PhpSpec 無法自動建立 Mock 物件的方法

編輯 `src/Song.php`

```php
    public function play()
    {
    }
```

```bash
t run
```

```
Do you want me to create `KK\Playlist::markAllAsPlayed()` for you? (y)
```

* Mock 物件沒有錯誤提示後， PhpSpec 就會繼續原來的自動建立測試對象方法的流程

編輯 `src/Playlist.php`

```php
    public function markAllAsPlayed()
    {
        foreach ($this->songs as $song) {
            $song->play();
        }
    }
```

```bash
t run
```

## 建立歌曲規格類別

```
t desc KK/Song
```

* 因為 `Song` 類別已經建立， PhpSpec 就不會問了

### 規格一：可以加星

編輯 `spec/SongSpec.php`

```php
    function it_can_be_stared()
    {
        $this->setStars(5);
        $this->getStars()->shouldBe(5);
    }
```

```bash
t run
```

```
Do you want me to create `KK\Song::setStars()` for you? (y)
```

```
Do you want me to create `KK\Song::getStars()` for you? (y)
```

編輯 `src/Song.php`

```php

class Song
{
    protected $stars;

    public function setStars($stars)
    {
        $this->stars = $stars;
    }

    public function getStars()
    {
        return $this->stars;
    }

    public function play()
    {
    }
}
```

* 加星功能即簡單的 setter / getter

### 規格二：不可加超過 5 的星

編輯 `spec/SongSpec.php`

```
    function its_stars_should_be_not_exceed_five()
    {
        $this->shouldThrow('InvalidArgumentException')->duringSetStars(8);
    }
```

* 當星數超過 5 的時候就要丟出異常

```bash
t run
```

編輯 `src/Song.php`

```php
    public function setStars($stars)
    {
        if ($stars > 5) {
            throw new InvalidArgumentException;
        }

        $this->stars = $stars;
    }
```

```bash
t run
```

### 重構

編輯 `src/Song.php`

```php
    public function setStars($stars)
    {
        $this->validateStarAmount($stars);

        $this->stars = $stars;
    }

    protected function validateStarAmount($stars)
    {
        if ($stars > 5) {
            throw new InvalidArgumentException;
        }
    }
```

* 讓程式碼具有可讀性

```bash
t run
```

## 規格三：可以被設定為已播放

編輯 `spec/SongSpec.php`

```php
    function it_can_be_marked_as_played()
    {
        $this->play();
        $this->shouldBePlayed();
    }
```

```bash
t run
```

```
Do you want me to create `KK\Song::isPlayed()` for you?
```

編輯 `src/Song.php`

```php
    protected $played = false;

    public function play()
    {
        $this->played = true;
    }

    public function isPlayed()
    {
        return $this->played;
    }
```

## 規格四：可以取得歌曲名稱

編輯 `spec/SongSpec.php`

```php
    function it_can_fetch_the_name_of_the_song()
    {
        $this->getName()->shouldBe('La la la');
    }
```

* 不使用 getter ，改用 constructor

```bash
t run
```

```
Do you want me to create `KK\Song::__construct()` for you? (y)
```

編輯 `src/Song.php`

```php
    protected $name;

    public function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
```

```bash
t run
```

編輯 `spec/SongSpec.php`

```
    function let()
    {
        $this->beConstructedWith('La la la');
    }
```

* `let` 方法會在 spec 類別的每個測試執行前被呼叫
* 在 `let` 方法中用 `beConstructedWith` 來注入參數

```bash
t run
```
