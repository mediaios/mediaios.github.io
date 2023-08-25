---
layout: post
title: 语音合成器
description: AVSpeechSynthesizer 语音合成器
category: blog
tag: ios, Objective-C, Swift
---

## 语音合成器

语音合成器技术是IOS7退出的，可以实现无网络语音功能，支持多种语言。下面我们直接上代码来讲述语音合成器 AVSpeechSynthesizer 的用法。

	#import "MWViewController.h"
	#import "MWBubbleCell.h"
	
	@interface MWViewController () <AVSpeechSynthesizerDelegate>
	@property (nonatomic,strong) AVSpeechSynthesizer *synthesizer;
	@property (strong, nonatomic) NSArray *voices;
	@property (strong, nonatomic) NSMutableArray *speechStrings;
	@property (strong, nonatomic) NSMutableArray *speechContent;
	@end
	
	@implementation MWViewController
	
	- (AVSpeechSynthesizer *)synthesizer
	{
	    if (!_synthesizer) {
	        _synthesizer = [[AVSpeechSynthesizer alloc] init];
	    }
	    return _synthesizer;
	}
	
	- (NSArray *)voices
	{
	    if (!_voices) {
	        _voices = @[[AVSpeechSynthesisVoice voiceWithLanguage:@"en-US"],
	                    [AVSpeechSynthesisVoice voiceWithLanguage:@"zh-Hans-CN"]
	                    ];
	    }
	    return _voices;
	}
	
	- (NSMutableArray *)speechStrings
	{
	    if (!_speechStrings) {
	        _speechStrings = [NSMutableArray arrayWithArray:@[@"Hello boys,How are you?",
	                                                          @"你好。",
	                                                          @"What are you doing?",
	                                                          @"我正在看书。",
	                                                          @"What's your favorite book?",
	                                                          @"AVFoundation",
	                                                          @"What's your favorite feature?",
	                                                          @"我什么都喜欢，我想我深深爱上了它。",
	                                                          @"OK,Have fun!"
	                                                          ]];
	    }
	    return _speechStrings;
	}
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    
	    self.tableView.contentInset = UIEdgeInsetsMake(20.0f, 0.0f, 20.0f, 0.0f);
	    self.synthesizer.delegate = self;
	    self.speechContent = [NSMutableArray array];
	    [self beginConversation];
	}
	
	- (void)beginConversation {
	    for (NSUInteger i = 0; i < self.speechStrings.count; i++) {
	        AVSpeechUtterance *utterance =
	        [[AVSpeechUtterance alloc] initWithString:self.speechStrings[i]];
	        utterance.voice = self.voices[i % 2]; // 指定语音
	        utterance.rate = 0.3f;  // 朗诵速度
	        utterance.pitchMultiplier = 0.8f;
	        utterance.postUtteranceDelay = 0.1f;
	        [self.synthesizer speakUtterance:utterance];
	    }
	}
	
	- (void)didReceiveMemoryWarning {
	    [super didReceiveMemoryWarning];
	    // Dispose of any resources that can be recreated.
	}
	
	#pragma mark - Table view data source
	- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
	
	    return self.speechContent.count;
	}
	
	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
	{
	    NSString *identifier = indexPath.row % 2 == 0 ? @"YouCell" : @"AVFCell";
	    
	    MWBubbleCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier forIndexPath:indexPath];
	    cell.messageLabel.text = self.speechContent[indexPath.row];
	    return cell;
	}
	
	
	
	#pragma mark -AVSpeechSynthesizerDelegate
	- (void)speechSynthesizer:(AVSpeechSynthesizer *)synthesizer didStartSpeechUtterance:(AVSpeechUtterance *)utterance
	{
	    [self.speechContent addObject:utterance.speechString];
	    [self.tableView reloadData];
	    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:self.speechContent.count - 1 inSection:0];
	    [self.tableView scrollToRowAtIndexPath:indexPath atScrollPosition:UITableViewScrollPositionBottom animated:YES];
	}
	
	@end
	
完整的代码可以在此处下载：[AVSpeechSynthesizerDemo](https://github.com/MaxwellQi/LearnAVFoundation) 
