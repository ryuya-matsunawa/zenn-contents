---
title: "Kivyで作る数独自動解答アプリ①"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kivy", "Python", "数独"]
published: true
---

## はじめに
今回は、PythonのオープンライブラリのKivyを使用して、数独の自動解答アプリを作っていきたいと思います。
実装している中でKivyの記事が少なかったので、備忘録も兼ねて書いてみました。
最終的にはiosアプリとしてリリースまで持っていきたいです。

## リポジトリ作成
数独の画面用のリポジトリを作りましょう。その後、仮想環境を作成して有効化します。

```
mkdir kivy-sample
cd kivy-sample
python3 -m venv venv
source venv/bin/activate
```

## 各種インストール
念の為pipをアップグレード
```
venv/bin/python3 -m pip install --upgrade pip
```

requirements.txtを作成し、以下のパッケージを入力

```txt:requirements.txt
kivy
pygame
Cython
```

以下のコマンドでパッケージを一括インストール
``` 
pip3 install -r requirements.txt
```

## ファイル作成
KivyファイルとPythonのファイルを作成
```kv:main.kv
Label:
    text: "Hello World"
```

```py:main.py
#-*- coding: utf-8 -*-

from kivy.app import App

class Main(App):
    pass

if __name__ == '__main__':
    Main().run()
```

以下のコマンドを実行
```
python3 main.py
```

実行するとこちらのような画面が表示されます。
![](/images/hello_world.png)

## 数独の画面作成
まずは盤面を作ります。全体の盤面となるMainGrid、各ブロックごとのSubGrid、各マスごとのCellクラスを作成します。
次に、FloatLayoutを作成してMainGridの位置などを設定します。

```kv:main.kv
#:set padding 20

<MainGrid>:
    rows: 3
    cols: 3
    spacing: 15

<SubGrid>:
    rows: 3
    cols: 3

<Cell>:
    size_hint_x: 1
    size_hint_y: 1
    font_size: 30

FloatLayout:
    BoxLayout:
        pos_hint: {'center_x': .5, 'center_y': .5}
        size_hint: (None, None)
        center: root.center
        orientation: 'vertical'
        size: [min(root.width, root.height) - 2 * padding] * 2
        MainGrid:
            id: main_grid
            size_hint_y: 9
            center: root.center
            size: [min(root.width, root.height) - 2 * padding] * 2
```

main.kvを変えたので、合わせてmain.pyを変更します。
ソースコードの全体像はこちらです。

```py:main.py
#-*- coding: utf-8 -*-

from itertools import product
from math import ceil
from kivy.app import App
from kivy.uix.button import Button
from kivy.uix.gridlayout import GridLayout

class MainGrid(GridLayout):
    pass

class SubGrid(GridLayout):
    pass

class Cell(Button):

    DIC = {0: "*", 1: "1", 2: "2", 3: "3", 4: "4", 5: "5", 6: "6", 7: "7", 8: "8", 9: "9"}

    def __init__(self, **kwargs):
        super(Cell, self).__init__(**kwargs)
        self.text = Cell.DIC[0]

class Main(App):

    def build(self):
        main_grid = self.root.ids.main_grid
        for _ in product(range(3), repeat=2):
            sub_grid = SubGrid()
            for _ in product(range(3), repeat=2):
                cell = Cell()
                sub_grid.add_widget(cell)
            main_grid.add_widget(sub_grid)

if __name__ == '__main__':
    Main().run()
```

各マスの初期値として＊を入れるようにしています。
```py:main.py
class Cell(Button):

    DIC = {0: "*", 1: "1", 2: "2", 3: "3", 4: "4", 5: "5", 6: "6", 7: "7", 8: "8", 9: "9"}

    def __init__(self, **kwargs):
        super(Cell, self).__init__(**kwargs)
        self.text = Cell.DIC[0]
```

また、main.kvで作ったMainGridの中にSubGridやCellを入れます。
```py:main.py
def build(self):
    main_grid = self.root.ids.main_grid
    for _ in product(range(3), repeat=2):
        sub_grid = SubGrid()
        for _ in product(range(3), repeat=2):
            cell = Cell()
            sub_grid.add_widget(cell)
        main_grid.add_widget(sub_grid)
```
add_widgetは名前の通りwidgetを追加するメソッドになります。
これでmain_gridにsub_gridを9個、各sub_gridに9個ずつwidgetを追加します。

これで以下のように盤面が表示されます。
![](/images/maingrid.png)

このままだと数字が入力できないので、数字を入力できるようにします。

## 数字入力用のエリアを追加
数字入力用のエリアをKivyファイルに追加します。
```kv:main.kv
FloatLayout:
    BoxLayout:
        pos_hint: {'center_x': .5, 'center_y': .5}
        size_hint: (None, None)
        center: root.center
        orientation: 'vertical'
        size: [min(root.width, root.height) - 2 * padding] * 2
        MainGrid:
            id: main_grid
            size_hint_y: 9
            center: root.center
            size: [min(root.width, root.height) - 2 * padding] * 2
        <!-- 以下を追加 -->
        SubGrid:
            id: number_grid
            size_hint_x: .25
            size_hint_y: 2.5
            center: root.center
            padding: [0, 10, 0, 0]
```
1から9までの数字を初期値に入れます。
```py:main.py
def build(self):
    main_grid = self.root.ids.main_grid
    for _ in product(range(3), repeat=2):
        sub_grid = SubGrid()
        for _ in product(range(3), repeat=2):
            cell = Cell()
            sub_grid.add_widget(cell)
        main_grid.add_widget(sub_grid)
    number_grid = self.root.ids.number_grid
    # 以下を追加
    for i in range(9):
        cell = Cell()
        cell.text = Cell.DIC[i+1]
        number_grid.add_widget(cell)
```

これで以下のように盤面が表示されます。
![](/images/number.png)

数字を入力するにあたって、選択したマスを保持していないければいいけないので、Cellをクリックした時のメソッドを追加します。

```py:main.py
class Main(App):

    def build(self):
        main_grid = self.root.ids.main_grid
        for _ in product(range(3), repeat=2):
            sub_grid = SubGrid()
            for _ in product(range(3), repeat=2):
                cell = Cell()
                self.selected_cell = None # この行を追加
                cell.fbind("on_press", self.select_cell) # この行を追加
                sub_grid.add_widget(cell)
            main_grid.add_widget(sub_grid)
        number_grid = self.root.ids.number_grid
        for i in range(1, 10):
            cell = Cell()
            cell.text = Cell.DIC[i]
            cell.fbind("on_press", self.set_number) # この行を追加
            number_grid.add_widget(cell)
    
    # このメソッドを追加
    def select_cell(self, instance):
        if self.selected_cell:
            self.selected_cell.background_color = (1, 1, 1, 1)
        self.selected_cell = instance
        self.selected_cell.background_color = (0, 1, 0, 1)

    # このメソッドを追加
    def set_number(self, instance):
        if self.selected_cell:
            self.selected_cell.text = instance.text
```

fbindでon_press時のメソッドを設定します。
数独の各マスにはマス選択時のメソッドをセットし、数字入力用のマスにはクリックした数字を入れるメソッドをセットします。(数独のルールに合ってるかの判定は今回は割愛)

![](/images/insert.gif)

これで、問題を作るところまではできました。
次の記事では、作成した問題から解答を作成できるようにしていきます。

完成版のソースコード
```kv:main.kv
#:set padding 20

<MainGrid>:
    rows: 3
    cols: 3
    spacing: 15

<SubGrid>:
    rows: 3
    cols: 3

<Cell>:
    size_hint_x: 1
    size_hint_y: 1
    font_size: 30

FloatLayout:
    BoxLayout:
        pos_hint: {'center_x': .5, 'center_y': .5}
        size_hint: (None, None)
        center: root.center
        orientation: 'vertical'
        size: [min(root.width, root.height) - 2 * padding] * 2
        MainGrid:
            id: main_grid
            size_hint_y: 9
            center: root.center
            size: [min(root.width, root.height) - 2 * padding] * 2
        SubGrid:
            id: number_grid
            size_hint_x: .25
            size_hint_y: 2.5
            center: root.center
            padding: [0, 10, 0, 0]

```

```py:main.py
#-*- coding: utf-8 -*-

from itertools import product
from kivy.app import App
from kivy.uix.button import Button
from kivy.uix.gridlayout import GridLayout

class MainGrid(GridLayout):
    pass

class SubGrid(GridLayout):
    pass

class Cell(Button):

    DIC = {0: "*", 1: "1", 2: "2", 3: "3", 4: "4", 5: "5", 6: "6", 7: "7", 8: "8", 9: "9"}

    def __init__(self, **kwargs):
        super(Cell, self).__init__(**kwargs)
        self.text = Cell.DIC[0]

class Main(App):

    def build(self):
        main_grid = self.root.ids.main_grid
        for _ in product(range(3), repeat=2):
            sub_grid = SubGrid()
            for _ in product(range(3), repeat=2):
                cell = Cell()
                self.selected_cell = None
                cell.fbind("on_press", self.select_cell)
                sub_grid.add_widget(cell)
            main_grid.add_widget(sub_grid)
        number_grid = self.root.ids.number_grid
        for i in range(1, 10):
            cell = Cell()
            cell.text = Cell.DIC[i]
            cell.fbind("on_press", self.set_number)
            number_grid.add_widget(cell)
    
    def select_cell(self, instance):
        if self.selected_cell:
            self.selected_cell.background_color = (1, 1, 1, 1)
        self.selected_cell = instance
        self.selected_cell.background_color = (0, 1, 0, 1)

    def set_number(self, instance):
        if self.selected_cell:
            self.selected_cell.text = instance.text

if __name__ == '__main__':
    Main().run()
```

## 終わりに
最近GitHubに草を生やし続けるチャレンジを始めてみたのでよかったら覗いてみてください。
https://github.com/ryuya-matsunawa
