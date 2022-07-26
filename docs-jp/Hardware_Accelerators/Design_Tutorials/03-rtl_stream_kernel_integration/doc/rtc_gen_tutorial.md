<p align="right"><a href="../../../../../README.md">English</a> | <a>日本語</a></p>

# RTL カーネル (rtc\_gen) の作成

Vitis/Vivado RTL Kernel Wizard を使用すると、簡単に RTL ベースのカーネルを作成および開発して、ほかの HLS C または Vitis ビジョン ライブラリ ベースのカーネルを Vitis とシームレスに統合できます。このチュートリアルでは、rtc\_gen カーネルを作成する詳細な手順を示します。チュートリアルのターゲット プラットフォームには、Alveo U200 を使用します。

## 最上位デザインの仕様の決定

rtc\_gen カーネルは、フォント ライブラリをグローバル メモリからオンチップ SRAM に読み出し、AXI-Stream ポートを介してクロック画像を出力する必要があります。フォント ライブラリの読み出しは 1 回限りのジョブであり、通常の作業でグローバル メモリ帯域幅の要件はありません。このため、フォント読み出しポートに 32 ビット (データ バス) の AXI4 マスター インターフェイスを使用して、リソースを節約します。ダウンストリーム処理のパイプラインでは、XF\_NPP8 形式 (毎クロック サイクルで 8 ピクセル処理) を使用するため、AXI-Stream 幅を 64 ビットに選択し、AXI-Stream トランザクションごとに 8 ピクセルを転送できるようにします。制御レジスタには、XRT と互換性のある AXI スレーブ インターフェイスを使用するので、標準の OpenCL API を使用してカーネルをプログラムおよび制御できます。

このため、rtc\_gen カーネルの最上位デザイン仕様を次のように一般化します。

**バス インターフェイス**

+ 制御用の AXI4-Lite スレーブ インターフェイス
+ データ幅 32 ビット、アドレス幅 64 ビットのフォント データの読み込み用の AXI4-Lite マスター インターフェイス
+ データ幅 64 ビットのクロック桁画像出力用の AXI-Stream マスター インターフェイス

**制御レジスタ**

| 番号| 引数| 幅| 説明
|----------|----------|----------|----------
| 0| work\_mode| 1| \[0]: カーネルの動作モードを決定します。<br> 0 = AXI 読み出しマスターを介してグローバル メモリからオンチップ SRAM までのフォントを読み込みます。<br>1 - AXI-Stream マスターを介して RTC 桁の図を出力します。
| 1| cs\_count| 32| \[21:0]: センチ秒カウンター。たとえば、システム クロックが 200MHz の場合、cs\_count は 2,000,000 に設定する必要があります。
| 2| time\_format| 1| \[0]: センチ秒が出力桁の画像に含まれているかどうかを確認します。<br> 0 - センチ秒の出力をディスエーブルにします。<br> 1 - センチ秒の出力をディスエーブルにします。
| 3| time\_set\_val| 32| 内部フリーランニング クロックの時間値を設定します。<br> \[31:24] - 時間 <br> \[23:16] - 分 <br> \[15:8] - 秒 <br> \[7:0] - センチ秒
| 4| time\_set\_en| 1| \[0]: このビットに 1 を書き込むと、time\_set\_value が内部フリーランニング クロックに読み込まれます。
| 5| time\_val| 32| 内部リアルタイム クロック時間値の読み取り専用レジスタ: <br> \[31:24] - 時間 <br> \[23:16] - 分 <br> \[15:8] - 秒 <br> \[7:0] - センチ秒
| 6| read\_addr| 64| AXI マスター ポインター。これは、フォント ライブラリ用の FPGA デバイス バッファー アドレスです。

<div align="center"><img src="./images/rtc_gen_block.png" alt="RTL カーネル" ></div>
<br/>

## RTL Kernel Wizard を使用したカーネル フレームの作成

では、RTL Kernel Wizard を使用して rtc\_gen カーネルのフレームを作成します。Vivado からウィザードを起動します。RTL カーネルおよび RTL Kernel Wizard の詳細なユーザー ガイドは、[Vitis アプリケーション アクセラレーション開発フローの資料](https://japan.xilinx.com/html_docs/xilinx2020_1/vitis_doc/devrtlkernel.html#qnk1504034323350)を参照してください。

git repo の ./rtc\_gen ディレクトリに移動し、vivado\_project という名前のディレクトリを作成してから、このディレクトリを入力します。

~~~
cd ./rtc_gen
mkdir vivado_project
cd ./vivado_project
vivado &
~~~

先ほど作成した vivado\_project ディレクトリに、rtc\_gen\_kernel という名前の新しい RTL プロジェクトを作成します。パーツを選択するページで、**\[Alveo U200 Data Center Accelerator Card]** を選択します。

<div align="center"><img src="./images/rtl_kernel_wiz_1.png" alt="RTL カーネル" ></div>
<br/>
プロジェクトが作成されたら、Flow Navigator で \[IP Catalog] をクリックし、IP カタログの検索ボックスに「RTL Kernel」と入力し、RTL Kernel Wizard をダブルクリックしてウィザードを起動します。

次の図のように、RTL Kernel Wizard の \[General Settings] タブでカーネル名を rtc\_gen に設定し、カーネル ベンダーを **xilinx.com** に設定し、\[has reset] オプションの値を 1 に変更します。

<div align="center"><img src="./images/rtl_kernel_wiz_2.png" alt="RTL カーネル" ></div>

<br/>

[Scalars] タブでカーネル引数をデザイン仕様として設定します。次の図の [Control Register] の表を参照してください。 **read\_addr** レジスタは、AXI マスター ポインターのようにスカラー引数とは見なされないため、このタブで設定する必要はありません。ここでは引数の型として **uint** を使用していますが、これらのビットはすべて使用されない可能性があります。

<div align="center"><img src="./images/rtl_kernel_wiz_3.png" alt="RTL カーネル" ></div>

<br/>

[Global Memory] タブでデザイン仕様に従って AXI マスター インターフェイス パラメーターを設定します。AXI マスター インターフェイスに **fontread\_axi\_m** という名前を付け、幅を 4 バイト (32 ビット) に変更し、関連する引数名を **read\_addr** に設定します。次のスナップショットを参照してください。

<div align="center"><img src="./images/rtl_kernel_wiz_4.png" alt="RTL カーネル" ></div>

<br/>

\[Streaming Interfaces] タブで AXI4-Sttream インターフェイスの数を 1 に設定し、**dataout\_axis\_m** という名前にし、モードを **\[Master]** に設定し、幅を 8 バイト (64 ビット) に設定します。次のスナップショットを参照してください。

<div align="center"><img src="./images/rtl_kernel_wiz_5.png" alt="RTL カーネル" ></div>

<br/>

最後にウィザードのサマリ ページを確認し、\[OK] ボタンをクリックして RTL カーネルの最上位フレームワークを生成します。

<div align="center"><img src="./images/rtl_kernel_wiz_6.png" alt="RTL カーネル" ></div>

<br/>

次の \[Generate Output Products] ポップアップ ウィンドウでは、\[Skip] ボタンをクリックしてウィンドウを閉じます。これで、\[Sources] ビューの \[Design Sources] グループに rtc\_gen.xci ファイルが表示されるよになります。rtc\_gen.xci ファイルを右クリックし、\[Open IP Example Design] をクリックします。

<div align="center"><img src="./images/rtl_kernel_wiz_7.png" alt="RTL カーネル" ></div>

<br/>

\[Open IP Example Design] ポップアップ ウィンドウで \[OK] ボタンをクリックすると、rtc\_gen\_ex という名前の別のプロジェクトが ./vavado\_project ディレクトリに作成され、別の Vivado セッションで自動的に開きます。主な作業プロジェクトとして rtc\_gen\_ex を使用し、rtc\_gen カーネル開発を終了します。

<div align="center"><img src="./images/rtl_kernel_wiz_8.png" alt="RTL カーネル" ></div>

<br/>

rtc\_gen\_ex プロジェクトには、自動的に生成された Verilog/SystemVerilog ソース コード ファイルがいくつか表示されます。

~~~
rtc_gen_control_s_axi.v                 
rtc_gen_example_adder.v                
rtc_gen_example_axi_read_master.sv     
rtc_gen_example_axi_write_master.sv   
rtc_gen_example_counter.sv              
rtc_gen_example_number_generator.sv    
rtc_gen_example.sv
rtc_gen_example_vadd_axis.sv
rtc_gen_example_vadd.sv
rtc_gen_tb.sv
rtc_gen.v
~~~

\[Sources] ビューの \[Hierarchy] タブでは、HDL ファイル階層を確認できます。これでカーネル フレームワークの作成は終了です。

<div align="center"><img src="./images/rtl_kernel_wiz_9.png" alt="RTL カーネル" ></div>

<br/>

## rtc\_gen カーネルの開発

rtc\_gen カーネルでは、次の生成済み RTL ファイルを使用します。

+ **rtc\_gen\_control\_s\_axi.v**

  上位レベルのシステムおよび XRT への AXI Lite スレーブ インターフェイスで、すべてのカーネル引数とカーネル制御信号 (ap\_start、ap\_done、ap\_idle、ap\_ready) が含まれます。このモジュールを少し変更して、読み取り専用レジスタの time\_val を実現します。

+ **rtc\_gen\_example\_axi\_read\_master.sv**

  単純な AXI Lite 読み出しマスターで、カーネルから直接呼び出すことができます。4 つの制御信号を使用してマスターをトリガーし、データ読み出しタスクを完了します。

  + ctrl\_start: マスター ステート マシンを開始する単一のサイクル パルス
  + ctrl\_done: AXI 読み出しタスクの終了を示す単一のサイクル パルス
  + ctrl\_addr\_offset: AXI 読み出し操作のベース アドレス。この信号は、カーネル引数 read\_addr を使用すると駆動できます。
  + ctrl\_xfer\_size\_in\_bytes: AXI バスから読み出されるバイト数。

  マスターの例もデータ転送の読み出しに AXI-Stream ポートを使用します。このポートは、データ パイプラインのように FIFO に簡単に接続できます。

+ **rtc\_gen\_example\_counter.sv**

  AXI 読み出しマスターの例のサブモジュール。

+ **rtc\_gen.v** すべてのサブモジュールをインスタンシエートする最上位のカーネル ラッパーの例です。このモジュールを変更して、rtc\_gen カーネルを構築します。

+ **rt\_gen\_tb.sv** ザイリンクス AXI Verification IP を使用したテストベンチの例です。このテストベンチを変更して rtc\_gen カーネルをテストできます。

AXI 読み出しマスターの接続の詳細は、これらの 5 つのファイルに加えて、rtc\_gen\_example\_vadd.sv を参照することもできます。AXI-Stream ポートの場合は単純なので、参照用の例は必要ありません。rtc\_gen カーネルの場合、Verilog ファイルの rtc\_gen\_core.v が作成され、カーネルのコア ファンクションを完了します。rtc\_gen\_core のファンクション図を次に示します。

<div align="center"><img src="./images/rtl_kernel_wiz_10.png" alt="RTL カーネル" ></div>

<br/>

コアのソース コード ディレクトリをクリーンにするため、生成した RTL ファイル、新しく作成した RTL ファイル、または変更した RTL ファイルを ~/rtc_gen/src ディレクトリにすべて配置します。これらのソース コード ファイルのディレクトリを確認してください。SPSR.v は、FPGA BlockRAM に合成できるパラメーター変更可能な SRAM テンプレートです。

~~~
./rtc_gen/src/rtc_gen_control_s_axi.v
./rtc_gen/src/rtc_gen_core.v
./rtc_gen/src/rtc_gen_example_axi_read_master.sv
./rtc_gen/src/rtc_gen_example_counter.sv
./rtc_gen/src/rtc_gen_tb.sv
./rtc_gen/src/rtc_gen.v
./rtc_gen/src/SPSR.v
~~~

次に、Vivado の rtc\_gen\_ex プロジェクトから既存のすべての Verilog/SystemVerilog ソース コード (IP グループ内のものを除く) を削除し、./rtc\_gen/src のファイルをプロジェクト (\[Simulation-Only Sources] の場合は rtc\_gen\_tb.sv、\[Design Sources] の場合はその他のファイル) を追加します。デザイン階層は次の図のようになります。rtc\_gen カーネルのコーディングはこれで終了です。

<div align="center"><img src="./images/rtl_kernel_wiz_11.png" alt="RTL カーネル" ></div>

<br/>

これで、標準的な Vivado デザイン手法を使用して、デザインをシミュレーションし、通常の RTL デザイン フローを実行できるようになりました。提供されるテキスト形式のフォント ライブラリ ファイル (./rtc\_gen ディレクトリの font\_sim\_data.txt) を rtc\_gen\_tb.sv テストベンチで読み込むと、シミュレーションできます。シミュレーション実行の場合は、これを ./rtc\_gen/vivado\_project/rtc\_gen\_ex/rtc\_gen\_ex.sim/sim\_1/behav/xsim/ にコピーします。

## RTL カーネルのパッケージ

デザインに問題がないことを確認したら、\[Flow] メニューから \[Generate RTL Kernel] をクリックし、ポップアップ ウィンドウで \[Sources-only kernel] を選択し、\[OK] ボタンをクリックしたら、RTL カーネル (rtc\_gen) の作成は終了です。

<div align="center"><img src="./images/rtl_kernel_wiz_12.png" alt="RTL カーネル" ></div>

<br/>

生成されたカーネル ファイルは ./rtc\_gen/vivado\_project/rtc\_gen\_ex/rtc\_gen.xo で、ダウンストリームの Vitis 統合フローで使用できます。Vivado の \[Tcl Console] ビューからは、Vivado が実際に次のコマンド ラインを使用してカーネルのパッケージを終了したことがわかります。

~~~
package_xo  -xo_path ./rtc_gen/vivado_project/rtc_gen_ex/exports/rtc_gen.xo \
            -kernel_name rtc_gen \
            -ip_directory ./rtc_gen/vivado_project/rtc_gen_ex/rtc_gen \
            -kernel_xml ./rtc_gen/vivado_project/rtc_gen_ex/imports/kernel.xml
~~~

./rtc\_gen/vivado\_project/rtc\_gen\_ex/rtc\_gen は rtc\_gen IP のフォルダーで、./rtc\_gen/vivado\_project/rtc\_gen\_ex/imports/kernel.xml はカーネル記述ファイルです。これらをスタンドアロンのディレクトリにコピーし、上記のコマンド ラインを使うと、カーネルをパッケージできます。これは、./hw/rtc\_gen\_ip ディレクトリと ./hw/rtc\_gen\_kernel.xml ファイルの元にもなります。


---------------------------------------


<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
<p align= center class="sphinxhide"><b><a href="../../../../README.md">メイン ページに戻る</a> &mdash; <a href="../../../README.md">ハードウェア アクセラレータ チュートリアルの初めに戻る</a></b></p></br>
<p align="center"><sup>この資料は 2021 年 2 月 8 日時点の表記バージョンの英語版を翻訳したもので、内容に相違が生じる場合には原文を優先します。資料によっては英語版の更新に対応していないものがあります。
日本語版は参考用としてご使用の上、最新情報につきましては、必ず最新英語版をご参照ください。</sup></p>
