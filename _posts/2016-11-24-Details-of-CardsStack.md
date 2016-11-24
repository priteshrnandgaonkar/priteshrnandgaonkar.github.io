---
layout: post
title: Details of CardsStack
---

CardsStack is an open source library which makes your `UICollectionView` to awesome set of cards using `UICollectionViewLayout`.

![Basic Interaction](https://priteshrnandgaonkar.github.io/assets/Gifs/BasicCardStackInteraction.gif) 

Lets understand the main components of the `CardsStack` library in detail and how its built. But before that, checkout the [library](https://github.com/priteshrnandgaonkar/CardsStack)

## CardLayout

This class is derived from `UICollectionViewLayout` and it is responsible for calculating the frame of the `UICollectionViewCell` which makes the card overlap over one another.

Before we delve into the details of this class, lets think, what we need as information to calculate the frame of the cards(`UICollectionViewCell`).

* Basic card UI info like, leading, trailing, width and height.
* CardOffset - offset between the cards when stacked.
* Collapsed and expanded height of the `UICollectionView`.
* Upward and downward threshold, so that if the card crosses the threshold it is either Collapsed or Expanded.

All the above info is contained in `Configuration` object. Along with this information, `CardLayout` also needs the amount by which the collection view is moved through pan gesture(discussed in next section), so that it can accordinly change the frame of cell.

To utilise this information and give the custom layout attributes to the cards, its important to understand how the `UICollectionView` coordinates with `UICollectionViewLayout`. 

Whenever the cardlayout of the `UICollectionView` is invalidated `UICollectionView` calls the following methods in the given order

* `func prepare()`
* `func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]?` (called manytimes)
* `func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes?`

`prepare()` is meant to be utilised for preparing the layout process by performing calculation which can be used when the system calls the second function. So I calculated the the attributes of the cell in `prepare()` based on the cards state and the above discussed information and stored it in `cachedAttributes` which is an array of `UICollectionViewLayoutAttributes`

System calls `func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]?` many times and passes the rect as a parameter and expects the attributes of the cell overlapping the given the rect.

Since we are clear with the understanding of the layout process, lets look at the frame calculation.

### Cards frame calculation

The frame of the card is dependent on its index in the stack and obviously on its UI information. So lets write a function which takes this parameters and returns the frame of the card.

``` swift
func frameFor(index: Int, cardState: CardState, translation: Float) -> CGRect {
    var frame = CGRect(origin: CGPoint(x: CGFloat(delegate.configuration.leftSpacing), y:0), size: CGSize(width: UIScreen.main.bounds.width - CGFloat(delegate.configuration.leftSpacing + delegate.configuration.rightSpacing), height: CGFloat(delegate.configuration.cardHeight)))
    var frameOrigin = frame.origin
    switch cardState {
    case .Expanded:
        let val = (delegate.configuration.cardHeight * Float(index))
        frameOrigin.y = CGFloat(Float(delegate.configuration.verticalSpacing * Float(index)) + val)
        
    case .InTransit:
        if index > 0 {
            
            let collapsedY = delegate.configuration.verticalSpacing + (delegate.configuration.cardOffset * Float(index))
            let finalDistToMove = Swift.abs(((delegate.configuration.verticalSpacing + delegate.configuration.cardHeight) * Float(index)) - collapsedY)
            let fract = (finalDistToMove * translation)/(delegate.configuration.expandedHeight - delegate.configuration.collapsedHeight)
            let val = CGFloat(delegate.configuration.verticalSpacing + (delegate.configuration.cardOffset * Float(index)) + fract)
            frameOrigin.y = val
        }
        
    case .Collapsed:
        if index > 0 {
            frameOrigin.y = CGFloat(delegate.configuration.verticalSpacing + (delegate.configuration.cardOffset * Float(index)))
        }
    }
    frame.origin = frameOrigin
    return frame
}

```
The above function is the most important part of the library as it calculates the frame of the card as user interacts and which is what this library offers to its users.

Lets understand what the function does. The above function calculates the frame for three different cases, which are `Expanded`, `InTransit` and `Collapsed`

* For `Expanded` case, the frame calculation is the easiest, which is (cardHeight + `verticalSpacing`) * index, where `verticalSpacing` is the minimum spacing between the cards when cards are in expanded state.


* For `Collapsed` the frame calculation follows the same logic as above which  is (`verticalSpacing` + (cardOffset * index)) here `verticalSpacing` is added so that the first card is at `verticalSpacing` distance from its top.


* `InTransit` calculation is the most complicated one
	* As discussed above to calculate the frame of the card which is `InTransit` is dependent on the points moved by `UICollectionView` in y - direction.

	
	* The y - position of the card frame is always calculated with refrence to y - position in the collapsed state.So the above function calculates the amount to be added(say `fract`) to the collapsed card's y position to get the y-position of the card in current state


	*  `fract` is calculated by [unitary method](https://en.wikipedia.org/wiki/Unitary_method)
		*  (Total distance `UICollectionView` has to travel to expand) ----> (users current translation)
		*  (Total distance card has to move from collapsed state to expanded state) ----> (`fract`)
		*  The cross multiplication gives the value of `fract`

	* Add `fract` to the value of the collapsed card's y-position

We will give the layout attributes of the `UICollectionViewCell` to the system whenever it calls `func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]?`

``` swift
 override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {

    var layoutAttributes = [UICollectionViewLayoutAttributes]()
    
    for attributes in cachedAttributes {
        if attributes.frame.intersects(rect) {
            layoutAttributes.append(cachedAttributes[attributes.indexPath.item])
        }

    }
    return layoutAttributes
}

override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    return cachedAttributes[indexPath.item]
}
```
First function finds all the attributes which overlaps with given rect and returns the same.

To invalidate the `UICollectionView` on every bound change, return `true` in the folowing function.

``` swift
override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
    return true
}
```

This ends the description of `CardLayout`. In the following section I have discussed the role played by `CardsManager`.

## CardsManager

As name suggest this class is responsible for managing the cards. `CardsManager` manages `UIPanGesture`, `UITapGesture` and their respective callbacks.
It is initialised as follows,

``` swift

cardsManager = CardsManager(cardState: cardsState, configuration: configuration, collectionView: collectionView, heightConstraint: collectionViewHeight)
        
```

`CardsManager` is intialised with `CardState`, `Configuration`, `UICollectionView`, and `NSLayoutConstraint`(for moving the cards). Where `Configuration` holds the information related to the placement of the cards, like
`cardOffset`, `collapsedHeight`, `expandedHeight` etc.


``` swift
internal enum CardState {
    case Expanded
    case InTransit
    case Collapsed
}
```
The above enum tells the current state of the cards, i.e `Expanded`, `InTransit` or `Collapsed`. `InTransit` tells that the cards are currently in motion, mostly due to pan gesture.

CardsManager needs height constraint(`NSLayoutConstraint`) and `UICollectionView` object to move the cards and invalidate layout whenever required.

CardsManger listens to the events of pan gesture and accordinly sets the state(`CardState`) of card and invalidates layout. 

``` swift
func pannedCard(panGesture: UIPanGestureRecognizer) {  
    guard let collectionView = self.collectionView else {
            return
        }
    let translation = panGesture.translation(in: collectionView.superview!)
        
    let distanceMoved = translation.y          
	switch panGesture.state {
	case .changed:
	// Calculate the fraction moved
	
	    heightConstraint.constant -= distanceMoved
	    
	    heightConstraint.constant = Swift.min(heightConstraint.constant, CGFloat(self.configuration.expandedHeight))
	    heightConstraint.constant = Swift.max(heightConstraint.constant, CGFloat(self.configuration.collapsedHeight))
	    
	    self.cardState = .InTransit
	    self.fractionToMove = Float(heightConstraint.constant - CGFloat(self.configuration.collapsedHeight))
	    self.collectionView?.isScrollEnabled = false
	    
	    self.collectionView?.collectionViewLayout.invalidateLayout()
	    self.collectionView?.superview?.layoutIfNeeded()
	    
	case .cancelled:
	    fallthrough
	case .ended:
	    
	    // Card is either expanded or collapsed
	    // Update the card state
	}
}

```

`CardsManager` is the delegate of the `UICollectionView` and it triggers `UICollectionViewDelegate`'s major callbacks and also a custom callback denoting the state change of the cards.

``` swift
/// Delegate to get hooks to interaction over cards
@objc public protocol CardsManagerDelegate {
    
    @objc optional func cardsPositionChangedTo(position: CardsPosition)
    @objc optional func tappedOnCardsStack(cardsCollectionView: UICollectionView)
    @objc optional func cardsCollectionView(_ cardsCollectionView: UICollectionView, didSelectItemAt indexPath: IndexPath)
    @objc optional func cardsCollectionView(_ cardsCollectionView: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt indexPath: IndexPath)
}
```

## CardStack

Its a wrapper over `CardsManager` to avoid exposing any complexity and making the interface as simple as possible for the developers using this library.

``` swift
public class CardStack {

    weak public var delegate: CardsManagerDelegate?

    /// init
    /// uses config = Configuration(cardOffset: 40, collapsedHeight: 200, expandedHeight: 500, cardHeight: 200, downwardThreshold: 20, upwardThreshold: 20) in
    /// init(cardsState: .Collapsed, configuration: config, collectionView: nil, collectionViewHeight: nil)
    /// - returns: CardStack
    public convenience init()

    /// init
    ///
    /// - parameter cardsState:           Initial state of the cards
    /// - parameter configuration:        Instance of the Configuration, which holds the UI related information
    /// - parameter collectionView:       UICollectionView
    /// - parameter collectionViewHeight: NSLayoutConstraint, height constraint of the collectionview
    ///
    /// - returns: CardStack
    public init(cardsState: CardsStack.CardsPosition, configuration: CardsStack.Configuration, collectionView: UICollectionView?, collectionViewHeight: NSLayoutConstraint?)

    /// changeCardsPosition(to position: CardsPosition)
    /// This function can be called on CardStack to change the state of cardsStack. It can be used to programmatically change the states of the cards stack
    /// - parameter position: CardsPosition
    public func changeCardsPosition(to position: CardsStack.CardsPosition)
}

```
I feel the interface is concise and clear enough.

I covered the major details of the library. If anything is unclear, do comment and share your thougths.

